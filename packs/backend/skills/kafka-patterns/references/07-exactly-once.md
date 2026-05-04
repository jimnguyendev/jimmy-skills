# Exactly-Once Semantics

## Context

In an event-driven pipeline, duplicate processing happens because of network unreliability, consumer crashes, retries, and rebalances. Exactly-once is not a single feature — it requires combining mechanisms across the entire pipeline.

### Two flow types

```
Aggregation Flow: Topic A → [Stage] → Topic B     (Kafka-to-Kafka)
Sink Flow:        Topic B → [Stage] → DB           (Kafka-to-external, has side effects)
```

- **Aggregation Flow** (Kafka → Kafka): Use **Kafka Transactions** — this is a solved problem.
- **Sink Flow** (Kafka → DB/external): The common case. Requires application-level work. The rest of this document focuses here.

## Theory

```
Exactly-Once = At-Least-Once + Idempotent Processing
```

- **At-Least-Once:** Kafka can guarantee no message is lost (producer and consumer configuration).
- **Idempotent Processing:** The consumer must ensure that processing a message twice produces the same result as processing it once. Kafka does **not** provide this — it's the application's responsibility.

## At-least-once: don't lose messages

### Producer side

**Risks:** network failure when sending; broker crashes before replicating.

**Configuration:**

```properties
# Retry on transient failures (30 is a sensible minimum)
retries=30

# Wait for ALL in-sync replicas to acknowledge before considering the write successful
acks=all

# Deduplicate retries on the broker side + preserve ordering within a partition
enable.idempotence=true
```

With `acks=all`, the producer waits until the leader and all in-sync replicas have written the message. Combined with retries and idempotence, this ensures the message is durably stored even if a broker fails.

### Consumer side

**Risks:** committing offsets before processing (crash = message lost); using a random consumer group on each restart (skips unconsumed messages).

**Configuration (two options):**

```properties
# Option A: Auto-commit (simpler, wider duplicate window)
enable.auto.commit=true

# Option B: Manual commit (tighter control, recommended for sink flows)
enable.auto.commit=false
# Call consumer.commitSync() after processing each batch

# Use a FIXED consumer group — never generate a random group ID
group.id=my-service-consumer-group
```

Manual commit (`enable.auto.commit=false` + `commitSync()` after processing) gives tighter at-least-once semantics because the offset advances only after your code finishes. Auto-commit commits on a timer, so a slow batch may have its offset committed before processing completes — causing at-most-once behavior for that batch.

**The at-least-once duplicate scenario:**

```
Consumer processes message at offset 2
Consumer commits offset 3 (next to read)
Consumer processes message at offset 3
— Consumer crashes before committing offset 4 —
Consumer restarts, asks broker for current offset → 3
Consumer processes message at offset 3 AGAIN (duplicate)
```

This duplicate is unavoidable with at-least-once. The idempotent consumer pattern below handles it.

## Idempotent consumer: don't process duplicates

### Type 1: Naturally idempotent operations

If the operation is inherently idempotent (SET, UPSERT, PUT), no extra work is needed:

```sql
-- Running this twice produces the same result
UPDATE users SET name = 'Nam' WHERE id = 1;
```

### Type 2: Deduplication for non-idempotent operations

For operations like INSERT, balance adjustments, or sending emails, use a **unique constraint** on the message ID.

Every message must carry a unique `request_id` (UUID recommended). The consumer stores this ID alongside the business data. If a duplicate arrives, the unique constraint rejects it.

```sql
CREATE TABLE transactions (
    request_id      UUID PRIMARY KEY,   -- unique constraint prevents duplicates
    transaction_id  UUID NOT NULL,
    debit_account   VARCHAR NOT NULL,
    credit_account  VARCHAR NOT NULL,
    amount          BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- First insert: succeeds
INSERT INTO transactions (request_id, transaction_id, debit_account, credit_account, amount)
VALUES ('R1', 'T1', 'A0', 'A1', 34000);

-- Duplicate insert: fails with unique constraint violation → skip
INSERT INTO transactions (request_id, transaction_id, debit_account, credit_account, amount)
VALUES ('R1', 'T1', 'A0', 'A1', 34000);
-- ERROR: duplicate key value violates unique constraint
```

For operations that span multiple tables, wrap the INSERT (with the `request_id` check) and the business logic in the **same database transaction**.

## Checklist

- [ ] Producer: `retries` ≥ 30 (or `Integer.MAX_VALUE`)
- [ ] Producer: `acks=all`
- [ ] Producer: `enable.idempotence=true`
- [ ] Consumer: `enable.auto.commit=true` (simple) or `false` with manual `commitSync()` after processing (tighter)
- [ ] Consumer: fixed `group.id` (no random generation)
- [ ] Consumer: idempotent — either naturally idempotent operations or deduplication via unique constraint on `request_id`
