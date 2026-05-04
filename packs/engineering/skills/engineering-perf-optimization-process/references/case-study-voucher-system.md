# Case Study: 30M Voucher Distribution System

This case study illustrates the escalation ladder in practice — a real system that progressed through multiple optimization steps driven by profiling and metric proof at each stage.

> **Source:** Based on `github.com/huykn/distributed-cache` heavy-write-api POC. Refer to the repository for full implementation details.

## Context

- **Goal:** Distribute 30M vouchers across multiple pods at 50K req/s per pod
- **Constraint:** Each voucher claimed exactly once (no duplicates, no losses)
- **Architecture:** Worker pod distributes via Redis Pub/Sub → Read pods serve claims

## Architecture Overview

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│   MySQL DB   │────▶│      Worker          │────▶│   Read Pod 0     │
│  (vouchers)  │     │ • Index to Redis     │     │ • MinuteStore[60]│
└──────────────┘     │ • Distribute batches │     │ • Lock-free pop  │
                     │ • Process claims     │     │ • 50K req/s      │
                     └──────────┬───────────┘     └──────────────────┘
                                │                          ▲
                     Redis Pub/Sub (per-pod channels)      │
                                │                   Client Traffic
                                ▼
                     ┌──────────────────┐
                     │   Read Pod 1     │
                     │ • MinuteStore[60]│
                     │ • Lock-free pop  │
                     │ • 50K req/s      │
                     └──────────────────┘
```

## How the Escalation Ladder Applied

### Step 1: Fix the query → Batch indexing

**Problem:** 30M vouchers in MySQL. Sequential reads too slow for distribution.

**Action:** Worker batch-indexes vouchers from MySQL to Redis. Read pods never touch MySQL directly.

**Metric proof to move on:** Redis indexing complete, but serving 50K req/s per pod with Redis round-trip per request would require massive Redis infrastructure.

### Step 2: Redis cache → Per-pod distribution

**Problem:** Even with Redis, 50K req/s per pod hitting Redis for every claim is unsustainable.

**Action:** Worker pre-distributes voucher batches to each pod via Redis Pub/Sub per-pod channels. Each pod holds its own vouchers in-memory.

**Metric proof to move on:** Distribution works, but in-memory voucher access needs to be lock-free to hit 50K req/s.

### Step 3-4: L1 cache + singleflight → MinuteStore with lock-free pop

**Problem:** In-memory storage with mutex locking cannot sustain 50K concurrent pops/second.

**Action:** Time-bucketed MinuteStore design with `atomic.Value` for storage and `atomic.AddInt64` for lock-free sequential pop. Each minute has its own store; worker pre-distributes for the next minute.

```go
type MinuteStore struct {
    Vouchers   []atomic.Value  // Lock-free read/write
    CurrentIdx int64           // Atomic pop index
    Total      int64           // Total vouchers loaded
}

// Lock-free pop: unique index per goroutine
idx := atomic.AddInt64(&store.CurrentIdx, 1) - 1
if idx >= atomic.LoadInt64(&store.Total) { return } // exhausted
val := store.Vouchers[idx].Load()
store.Vouchers[idx].Store([]byte(nil)) // GC release
```

### Step 5: Zero-serialization → Snowflake ID + binary format

**Problem:** JSON marshal/unmarshal per voucher claim wastes CPU at 50K req/s.

**Action:** Snowflake ID encoding packs validity, voucher ID, and PIN into a single `uint64`. Binary voucher format uses null-separated bytes instead of JSON.

```
Snowflake: Bit 63 = valid flag | Bits 31-62 = voucher ID | Bits 0-30 = PIN
Bytes:     [code_bytes][null_separator][is_valid_byte]
```

Stateless validation: decode snowflake → extract valid bit → done. No DB query needed.

### Step 6: Lock-free patterns → Full pipeline

**Problem:** Claim processing (validation, history, persistence) on the hot path blocks throughput.

**Action:** Channel-based async pipeline separates hot path from heavy work:

```
HandleSpin() → ClaimQueue (chan, 50K buffer) → processClaimQueue()
    ↓                                              ↓
SpinHistoryQueue → bulk write to Redis     Validate snowflake
                   (every 1s or 1K batch)  → Publish claim
                                           → Worker persists
```

Hot path: atomic pop + channel send (nanoseconds). All I/O is background.

## Key Design Decisions

| Decision | Why | Alternative considered |
|---|---|---|
| Per-pod Pub/Sub channels | No broadcast waste; targeted distribution | Shared channel (all pods receive all batches) |
| 60 MinuteStores | Time-bucketed isolation; only active minutes allocate memory | Single shared buffer (contention + no isolation) |
| Lazy allocation | 60 \* 3M \* 16B = 2.88GB if pre-allocated | Pre-allocate all (wastes memory) |
| Snowflake ID encoding | Stateless validation without DB query | DB lookup per claim (too slow) |
| Unconsumed migration | Leftover vouchers recovered to future minutes | Discard leftovers (voucher loss) |
| Chaos engineering | Pod restart recovers state from Redis | No recovery (data loss on restart) |

## Lessons for the Escalation Ladder

1. **Each step was driven by a measured bottleneck**, not speculation. Redis was fast enough for distribution but not for 50K claim/s per pod — that's what drove the in-memory design.

2. **Zero-serialization saved CPU but required careful format design.** Snowflake encoding is elegant but adds complexity. Only justified because profiling showed marshal/unmarshal was a real bottleneck at target throughput.

3. **Lock-free patterns work when the access pattern fits.** Sequential pop with `atomic.AddInt64` is perfect for voucher distribution (each voucher claimed once). This pattern does NOT generalize to arbitrary key-value access.

4. **Async pipeline is the highest-leverage pattern.** Moving I/O off the hot path (channel-based claim processing, batched history writes) had the largest impact on sustained throughput.

5. **Rollback and recovery were designed in from the start.** Chaos engineering with pod kill/restart and Redis state recovery ensures the system is production-ready, not just benchmark-ready.
