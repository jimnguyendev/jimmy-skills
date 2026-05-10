# Lock-Free Patterns for High Throughput

When mutex contention or allocation pressure appears in the CPU/block profile and throughput targets exceed 10K req/s per instance, lock-free patterns eliminate synchronization bottlenecks. Apply these only when profiling confirms contention — premature lock-free code adds complexity without benefit.

> **Source:** Patterns extracted from `github.com/huykn/distributed-cache` heavy-write-api POC (30M voucher distribution, 50K req/s per pod). Refer to official Go documentation for `sync/atomic` and `sync` package semantics.

## Pattern 1: Atomic Operations Instead of Mutexes

Replace mutex-guarded state checks and counters with atomic operations on the hot path.

**Closed state check (read path guard):**

```go
// Hot path: no mutex, no contention
if atomic.LoadInt32(&sc.closed) != 0 {
    return nil, false
}
```

**Lock-free counter (metrics):**

```go
func (sc *SyncedCache) recordLocalHit() {
    atomic.AddInt64(&sc.stats.LocalHits, 1)
}
```

**Lock-free pop (unique index per goroutine):**

```go
idx := atomic.AddInt64(&store.CurrentIdx, 1) - 1
size := atomic.LoadInt64(&store.Total)
if idx >= size { return } // No items available
```

`atomic.AddInt64` returns a unique index per goroutine — no mutex needed for sequential access to a shared buffer.

**When to use:** Any counter, flag, or monotonic index on the hot path that currently uses `sync.Mutex`.

## Pattern 2: `atomic.Value` for Lock-Free Read/Write

`atomic.Value` provides lock-free reads with goroutine-safe writes for interface values. Useful when one writer updates shared data that many readers consume concurrently.

```go
type MinuteStore struct {
    Vouchers []atomic.Value // Each holds []byte
}

// Write (single writer per slot)
store.Vouchers[pos].Store(voucherBytes)

// Read (many concurrent readers)
val := store.Vouchers[idx].Load()
```

**When to use:** Shared data with single-writer/multi-reader pattern, or when storing pre-computed results that are periodically refreshed.

## Pattern 3: Singleflight for Cache Stampede Prevention

When a cache entry expires, N goroutines may simultaneously discover the miss. `singleflight.Group` deduplicates concurrent fetches for the same key.

```go
result, _, _ := sc.sfGroup.Do(key, func() (any, error) {
    // Double-check L1 (another goroutine may have populated it)
    if value, found := sc.local.Get(key); found {
        return value, nil
    }
    // Only ONE Redis call for N concurrent requests
    data, err := sc.store.Get(ctx, key)
    // ...
})
```

- Without singleflight: 1000 concurrent misses = 1000 Redis calls
- With singleflight: 1000 concurrent misses = 1 Redis call

→ See [Caching Patterns](caching.md) for comprehensive singleflight implementation with generics alternatives and full code examples.
→ See `jimmy-skills@backend-go-concurrency` for `singleflight` API details and `sync.Map` vs `RWMutex` guidance.

## Pattern 4: Channel-Based Async Processing

Separate the hot path (nanoseconds) from heavy work (validation, persistence) using non-blocking channel sends.

```go
// Hot path: non-blocking enqueue
select {
case p.ClaimQueue <- ClaimRequest{...}:
default: // Drop if full — preserves hot path latency
}

// Background worker: bulk processing
func (p *ReadPod) processClaimQueue(ctx context.Context) {
    for claim := range p.ClaimQueue {
        p.processSingleClaim(claim)
    }
}
```

**Key:** The hot path only does atomic pop + channel send. Heavy work (validation, Redis write, history persistence) happens in background goroutines.

**When to use:** Any hot path that currently does synchronous I/O (logging, metrics push, audit trail) that can tolerate async processing.

## Pattern 5: Lazy Allocation with Double-Check Locking

Defer expensive memory allocation until first use. Avoids wasting memory on resources that may never be accessed.

```go
func (p *MinuteStore) ensureAllocated() {
    if atomic.LoadInt32(&p.allocated) == 1 {
        return // Fast path: already allocated
    }
    p.allocMu.Lock()
    defer p.allocMu.Unlock()
    if p.allocated == 1 { return } // Double-check after lock
    p.Vouchers = make([]atomic.Value, bufCap)
    atomic.StoreInt32(&p.allocated, 1)
}
```

**Impact:** 60 time-buckets \* 3M slots \* 16 bytes = 2.88GB if all pre-allocated. Lazy allocation means only active buckets consume memory.

**When to use:** Large buffers or data structures that are partitioned by time, shard, or tenant where only a subset is active at any moment.

## Pattern 6: Batch Counter Updates

At extreme throughput (>50K ops/s), even atomic operations create cache-line contention across cores. Batch updates to reduce contention.

```go
localSpins := int64(0)
for {
    // ... do work ...
    localSpins++
    if localSpins%1000 == 0 {
        atomic.AddInt64(&totalSpins, localSpins)
        localSpins = 0
    }
}
```

Reduces atomic contention from N/1 to N/1000. Trade-off: counter is eventually consistent (up to 1000 ops stale).

**When to use:** Metrics counters at >10K ops/s where exact real-time accuracy is not required.

## Pattern 7: Memory Slot Clearing for GC

After consuming a value from an `atomic.Value` slot, explicitly clear the reference to allow GC to reclaim memory.

```go
// After reading voucher:
store.Vouchers[idx].Store([]byte(nil)) // Help GC release bytes
```

Without this, consumed data stays referenced in `atomic.Value`, preventing garbage collection of potentially gigabytes of consumed entries.

**When to use:** Any buffer or pool where entries are consumed once and the backing storage is long-lived.

## Pattern 8: `sync.Pool` for Allocation Reuse

Reuse short-lived structs across operations to reduce GC pressure on the hot path.

```go
var poolVoucherBatch = sync.Pool{
    New: func() any { return &VoucherBatch{} },
}

batch := poolVoucherBatch.Get().(*VoucherBatch)
defer poolVoucherBatch.Put(batch)
if err := json.Unmarshal(event.Value, &batch); err != nil {
    return err
}
```

**When to use:** High-allocation hot paths where profiling confirms GC pressure (`go tool pprof -alloc_objects`). Always benchmark — `sync.Pool` adds complexity and only helps when allocation rate is high enough.

**Critical rules:** Reset struct state before `Put()` to avoid data leaks. Return copies of pooled data, not references (callers may hold them past `Put()`). Don't pool objects >32KB or infrequently used objects. See [Memory Optimization](memory.md) for full `sync.Pool` best-practice rules.

→ See `jimmy-skills@backend-go-concurrency` for `sync.Pool` API details and lifecycle considerations.

## Decision Summary

| Pattern | Trigger (from profile) | Complexity |
|---|---|---|
| 1. Atomic ops | Mutex in hot path CPU profile | Low |
| 2. atomic.Value | RWMutex contention on shared data | Low |
| 3. Singleflight | Cache stampede on cache miss | Low |
| 4. Channel async | Synchronous I/O on hot path | Medium |
| 5. Lazy allocation | Large upfront memory allocation | Medium |
| 6. Batch counters | Atomic contention at >50K ops/s | Medium |
| 7. Slot clearing | Memory not reclaimed after consumption | Low |
| 8. sync.Pool | High alloc_objects in heap profile | Medium |
