# Message Ordering

> "There are only two hard problems in distributed systems: 2. Exactly-once delivery 1. Guaranteed order of messages 2. Exactly-once delivery" — Mathias Verraes

## Why ordering matters

- **Gaming:** the sequence move → jump → turn left determines game state.
- **Order processing:** placed → paid → shipped → delivered is a mandatory lifecycle.
- **Financial systems:** debit before credit, not the reverse.

### Don't use physical clocks

Physical clocks are unreliable for ordering in distributed systems (clock skew, NTP drift). Use Kafka partition offsets, sequence numbers, or logical clocks (Lamport/Vector clocks) instead. See: "Ordering Distributed Events" by Vaidehi Joshi.

## Total order vs. partial order

**Total order:** the exact sequence of every event is known. Kafka guarantees total order **only within a single partition**.

**Partial order:** the order within related groups is known, but the order between groups is not. In a distributed system without a global clock, events can only be partially ordered.

> **Most systems only need partial order.** A Pac-Man game doesn't need to order Player 1's actions relative to Player 2's — only each player's actions need to be ordered internally.

## How Kafka provides ordering

Each partition is an ordered, immutable log. A consumer reading from one partition sees messages in the exact order they were written.

A topic with multiple partitions provides partial order: each partition is totally ordered, but there's no ordering guarantee across partitions.

```
Topic X:
  Partition 0: [0, 4, 6]  → Consumer 0 reads: 0, 4, 6  (ordered)
  Partition 1: [1, 3, 7]  → Consumer 1 reads: 1, 3, 7  (ordered)
  Partition 2: [2, 5]     → Consumer 2 reads: 2, 5      (ordered)
  — no ordering between partitions —
```

## Grouping related messages: use keys

Messages with the **same key** are routed to the **same partition** by the producer's default partitioner (murmur2 hash of the key, mod partition count).

Choose the key based on what needs to be ordered together:

- Order events → key = `order_id`
- User actions → key = `user_id`
- Account transactions → key = `account_id`

> **Gotcha:** If the partition count changes, the hash mapping changes. Messages with the same key may start going to a different partition. This is why over-provisioning partitions upfront matters — see `references/01-topic-width.md`.

## End-to-end message ordering

To guarantee ordering from producer to consumer:

1. **Enable producer idempotence:** `enable.idempotence=true` — ensures that retries don't reorder messages within a partition.
2. **Control in-flight requests:** With idempotence enabled, Kafka guarantees ordering for up to 5 in-flight requests per connection. Without idempotence, you **must** set `max.in.flight.requests.per.connection=1` to prevent reordering on retries.
3. **Use a key** to route related messages to the same partition.
4. **Each partition is consumed by exactly one consumer** in a consumer group — Kafka guarantees this.

```properties
enable.idempotence=true                    # preserves ordering on retry
max.in.flight.requests.per.connection=5    # safe with idempotence (set to 1 without it)
```

This gives you: Producer (ordered by key) → Partition (total order) → Consumer (reads in partition order).

## Cross-partition mutual exclusion

Sometimes two messages for the same logical entity land on different partitions (due to repartitioning, cross-topic flows, or key changes). The requirement is: **only one should be processed at a time** to ensure data integrity.

This is not an ordering problem — it's a **mutual exclusion** problem. The solution is **distributed locking**.

```
Consumer 0 (Partition 0) ──[acquire lock: account-123]──→ Lock Manager
Consumer 1 (Partition 1) ──[acquire lock: account-123]──→ [ZooKeeper/Redis]
                                                              │
Consumer 0 ←── lock acquired ─────────────────────────────────┘
Consumer 1 ←── lock denied (retry or skip) ───────────────────┘
```

### Distributed locking considerations

| Concern | Guidance |
|---------|---------|
| **Lock key granularity** | Too coarse → high contention, low throughput. Too fine → overhead, many keys. |
| **Lock failure** | What happens when the lock service is down? Have a fallback or circuit-breaker strategy. |
| **Deadlock prevention** | Always set a TTL on locks. Implement lock timeout on the client side. |
| **Performance** | Each message requires 2 network round-trips (acquire + release). Batch where possible. |

**Tools:** Redis/Redlock (fast, widely adopted, sufficient for most use cases), etcd (strongly consistent, good for critical locking). Avoid ZooKeeper for new application-level locking — it is being phased out of the Kafka ecosystem (KRaft mode) and adds operational burden.

## Checklist

- [ ] Determine whether you need total order or partial order (usually partial suffices).
- [ ] Use **keys** to group related messages into the same partition.
- [ ] Set `enable.idempotence=true` on the producer to prevent reordering during retries.
- [ ] Verify `max.in.flight.requests.per.connection` ≤ 5 with idempotence, or = 1 without it.
- [ ] If cross-partition mutual exclusion is required, implement distributed locking.
- [ ] **Never** use physical wall-clock timestamps to determine event order in a distributed system.
