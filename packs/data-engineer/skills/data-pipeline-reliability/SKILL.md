---
name: data-pipeline-reliability
description: Use this skill when the user needs reliable ingestion or transformation pipelines, especially for retries, duplicates, idempotency, backfills, CDC, incremental loads, overwrites, merges, transactions, or ordered delivery. Apply it when the task is to make a data pipeline safe to rerun or recover, even if the user does not explicitly say "idempotency" or "reliability." For warehouse layering, semantic metrics, or mart design, prefer `data-engineering`. For RDW vs lakehouse or maturity-based architecture choice, prefer `data-architecture-strategy`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Pipeline Reliability

Use this skill when the main question is whether a pipeline can be rerun, retried, backfilled, or recovered without corrupting data.

## Boundaries

- Use `data-engineering` when the main decision is platform shape, marts, semantic metrics, or serving layers.
- Use `data-architecture-strategy` when the main decision is which architecture family to adopt.
- Use `data-quality` when the main concern is whether outputs are trustworthy once they are produced.

## Default procedure

1. Identify the pipeline mode:
   - full refresh
   - incremental batch
   - CDC or streaming
   - backfill or replay
2. Define the failure mode to guard against:
   - duplicate writes
   - partial writes
   - inconsistent backfill
   - out-of-order delivery
3. Pick the simplest idempotency mechanism that matches the write path.
4. Make the retry and backfill behavior explicit before implementing the pipeline.

## Defaults

- Use batch before streaming unless latency requirements are explicit.
- For full refreshes, prefer partition-level overwrite or metadata-level replacement over row-by-row deletes.
- For incremental changes, use merge semantics only when immutable keys are available.
- For backfills, assume a plain merge is not enough unless state restore or version-aware logic exists.
- For dbt-style ELT, rely on transaction scope for atomic model writes, but do not confuse that with full backfill safety.

## Pattern guide

### Ingestion patterns (8)

| Pattern | When to use |
|---|---|
| Full Loader | Initial sync, small reference tables, zero state needed |
| Incremental Loader | Routine batch on large datasets — delta column or time-partitioned |
| CDC (Debezium / Delta CDF) | Sub-minute latency or delete capture from RDBMS |
| Passthrough Replicator | Cross-environment copy, no PII, preserve original format |
| Transformation Replicator | Cross-environment copy with PII removal or masking |
| Compactor | Too many small files from streaming ingestion |
| Readiness Marker | Downstream must not read incomplete data (`_SUCCESS` flag or partition convention) |
| External Trigger | Data arrives unpredictably; polling wastes compute |

### Idempotency patterns (7)

| Family | Pattern | When to use |
|---|---|---|
| **Overwrite** | Fast Metadata Cleaner | Full dataset, partitioned target, metadata ops available (TRUNCATE/DROP) |
| **Overwrite** | Data Overwrite | Full dataset, object store or no metadata ops (INSERT OVERWRITE, partition replace) |
| **Merge** | Merger | Incremental changes with stable immutable keys, no backfill risk (MERGE/UPSERT) |
| **Merge** | Stateful Merger | Incremental changes where backfill consistency matters (version tracking + restore before merge) |
| **Database-layer** | Keyed Idempotency | Streaming writes to key-value store — derive key from append time, not event time |
| **Database-layer** | Transactional Writer | Multi-statement warehouse writes that must appear atomically |
| **Database-layer** | Proxy | Audit trail required — immutable internal tables + VIEW pointing to latest |

### Ordering patterns (2)

- Partial-commit APIs where ordered bulk writes matter: Bin Pack Orderer
- Simple low-volume sequential processing: FIFO Orderer

## Gotchas

- A job that is safe to retry is not automatically safe to backfill.
- Event time is often a bad idempotency key when late data can arrive; append or ingestion time is more stable.
- A plain merge can produce inconsistent state during replay if future data is already present.
- Ordering and exactly-once are separate problems.

## References

- Read [references/reliability-patterns.md](references/reliability-patterns.md) when the task involves overwrite vs merge, replay behavior, backfill consistency, or ordering guarantees.
