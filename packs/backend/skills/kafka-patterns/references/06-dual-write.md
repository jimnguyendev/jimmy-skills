# Dual Write Problem

> "Everything in software architecture is a trade-off, and the why is more important than how." — Neal Ford

## Problem types

A process needs to write to two systems atomically:

| Type | Description | Solution |
|------|-------------|----------|
| **United** | Both writes go to Kafka (Topic A + Topic B) | Kafka Transaction |
| **Separate** | One write to a database, one to Kafka | Transactional Outbox Pattern |

## United dual-write → Kafka Transaction

When both writes target Kafka topics on the same cluster, use **Kafka Transactions** (available since release 0.11.0). The producer wraps both writes in `beginTransaction()` / `commitTransaction()`, and either both succeed or both are rolled back.

Be aware of Kafka Transaction limitations: they only work within a single Kafka cluster, add latency, and require `isolation.level=read_committed` on consumers.

## Separate dual-write — the core problem

The common case: a handler saves state to a database **and** emits an event to Kafka.

**Why this is hard:** there is no transaction that spans both a database and a message broker. You cannot atomically commit to PostgreSQL and produce to Kafka.

### Failure scenario

```
1. Save state to DB  → ✅ succeeds
2. — crash or network failure here —
3. Emit event to Kafka → ❌ never executes
```

Result: state is updated but the event is never emitted. The frequency is rare, but the **impact is severe** (e.g., money debited but payment event never sent) and **hard to detect and resolve**.

### Failed attempts

**Attempt 1 — Emit first, save second:**
Reverses the inconsistency (event emitted, state not saved). Equally broken.

**Attempt 2 — Wrap both in a DB transaction:**
The Kafka produce call executes **outside** the DB transaction boundary. If the produce succeeds but a subsequent error causes a rollback, the event is already on the topic. The DB rolls back, Kafka does not.

## Potential solutions

| Pattern | Assessment |
|---------|-----------|
| Listen to Yourself | Introduces new consistency problems. Impractical. |
| Event Sourcing | Complex, hard to handle errors, not straightforward for most use cases. |
| Change Data Capture (CDC) | Simple, but you lose control over the event format — the connector captures DB changes, not domain events. |
| **Transactional Outbox** | Straightforward, effective. **Recommended.** |

## Transactional Outbox Pattern (recommended)

### How it works

1. The handler writes **both** the state change and the event into the **same database transaction**: state goes to the business table, the event goes to an **outbox table**.
2. A **CDC connector** (Debezium) reads the database WAL (Write-Ahead Log) and detects new rows in the outbox table.
3. The connector **emits** the event to Kafka.

Because both writes happen in one DB transaction, they either both commit or both roll back — the inconsistency window is eliminated.

### Outbox table schema

```sql
CREATE TABLE outbox (
    id         TEXT        PRIMARY KEY,
    topic      VARCHAR     NOT NULL,
    header     BYTEA,
    key        BYTEA,
    value      BYTEA,
    created_at TIMESTAMP   DEFAULT NOW()
);
```

Store **pre-serialized messages** (already encoded with your serialization format) so the CDC connector can emit them directly without transformation.

### Implementation stack

- **CDC connector:** Debezium (reads PostgreSQL WAL via logical replication)
- **Deployment:** Kafka Connect (hosts the Debezium connector)
- **Kubernetes management:** Strimzi (operator for Kafka and Kafka Connect)

### Gotchas

- **Outbox table growth:** The table grows indefinitely. Set up a cleanup job (delete rows older than N hours/days after they've been emitted). Without this, DB performance degrades.
- **Message ordering:** CDC reads the WAL in commit order, which preserves ordering within a single transaction. Concurrent transactions may interleave — if strict ordering matters, use a single-threaded writer or a sequence column.
- **At-least-once delivery:** If the CDC connector crashes after emitting but before checkpointing, it will re-emit on restart. Downstream consumers **must be idempotent** (see `references/07-exactly-once.md`).
- **Latency:** Adds one hop (DB WAL → CDC → Kafka) compared to direct produce. Typical latency is sub-second but depends on CDC polling interval.
