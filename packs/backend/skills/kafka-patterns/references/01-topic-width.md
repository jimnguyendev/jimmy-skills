# Topic Width (Partition Sizing)

## Context

Choosing the right number of partitions is a one-way decision with lasting consequences:

- Kafka **only supports increasing** the partition count — no decreasing.
- Adding partitions doesn't interrupt clients, but **the producer hash is not consistent**: after adding partitions, records with the same key may route to a different partition.
- The maximum number of consumers in a group equals the number of partitions.
- Over-provisioning is common but adds load on brokers and consumers.

## The formula

```
Partitions = Expected Throughput / Throughput per Partition
```

Where `Throughput = max(Write Throughput, Read Throughput)`.

Pick a **divisible number** (e.g., 6, 8, 12, 16, 24, 32) — it distributes evenly when scaling consumers.

### Confluent limits

- Per broker: up to **4,000 partitions**
- Per cluster: up to **200,000 partitions**

## Sizing by write throughput

Best for write-heavy workloads: IoT events, user behavior tracking, logging.

```
Expected Throughput = QPS × Message Size
Write Throughput per Partition ≈ 10 MB/s
Message Size ≈ 700 bytes raw, ≈ 500 bytes compressed
```

| Load level | Messages/s | Recommended partitions |
|-----------|-----------|----------------------|
| Low | < 10K | < 4 |
| Medium | 10K – 100K | 8 – 16 |
| High | 100K – 1M | 20 – 32 |

## Sizing by read throughput

Best for process-heavy workloads: payments, order processing, transformations.

```
Expected Throughput = Expected TPS (transactions per second)
Read Throughput per Partition = depends on app logic and DB → must load-test
```

| Load level | Messages/s | Recommended partitions |
|-----------|-----------|----------------------|
| Low | < 1K | ≤ 8 |
| Medium | 1K – 10K | 8 – 16 |

> Adding more partitions **may not improve throughput** if the bottleneck is consumer processing speed, not Kafka delivery.

## Decreasing partitions (workaround)

Kafka doesn't support reducing partitions directly. Use this migration procedure:

1. Create a **new topic** with the desired partition count.
2. Use **MirrorMaker** to replicate from old topic → new topic.
3. When replication catches up, **stop old producers** briefly for parity.
4. Switch producers to **double-publish** to both old and new topics.
5. Reset consumer offsets on the new topic:
   - `auto.offset.reset=latest` — consumers must be idempotent (will skip in-flight messages).
   - `auto.offset.reset=earliest` — reset offsets to the cutoff timestamp.

> Consumers **must be idempotent** during the migration to handle potential duplicates.
