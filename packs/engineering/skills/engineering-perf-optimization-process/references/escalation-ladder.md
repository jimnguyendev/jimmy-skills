# Escalation Ladder

Start at step 1. Move to the next step ONLY when the current step is insufficient AND you have metric proof.

## Step 1: Fix the query

**When:** Database queries appear in the profile as the dominant cost.

**Actions:**

- Add missing indexes (verify with EXPLAIN)
- Replace N+1 fetch loops with batched queries (`WHERE id = ANY($1)`)
- Use prepared statements for hot-path queries
- Add strict query timeout for hot path (e.g., 50ms) with fallback to stale cache or 503
- Enable connection pooling with bounded pool size

**Metric proof to escalate:** Queries are already optimized (index-only scans, no N+1) but latency target is still not met. Show EXPLAIN output and query latency histogram.

**Rollback:** Indexes can be dropped. Prepared statements can be reverted. Timeout fallback uses feature flag.

## Step 2: Add single-layer cache (Redis)

**When:** Repeated reads of the same data dominate the profile, and the data tolerates staleness.

**Actions:**

- Cache hot-path query results in Redis with TTL
- Add TTL jitter (e.g., base TTL +/- 10%) to avoid synchronized expirations
- Define key format upfront (e.g., `item:{id}`, `feed:{user_id}:{bucket}`)
- Invalidate on write for critical fields only (event-driven, not TTL-only)
- Add cache hit rate metrics (must monitor from day one)

**Metric proof to escalate:** Cache hit rate is high (>90%) but Redis round-trip latency (typically 1-3ms) still accounts for a significant portion of p99. Show cache hit rate + Redis latency histogram.

**Rollback:** Feature flag to bypass cache and read directly from DB.

## Step 3: Add in-process L1 cache

**When:** Redis round-trip is the bottleneck despite high hit rate. Data access is read-heavy with predictable hot keys.

**Actions:**

- Add in-process LRU or LFU cache as L1 (e.g., Ristretto, golang-lru)
- L1 serves hot keys in ~100ns vs Redis ~1-3ms
- Keep L1 TTL short (seconds) or use pub/sub invalidation for consistency
- Size L1 to fit in memory budget (Gate 1) — do not cache everything
- Prevent stale write-back: reader nodes should not write to Redis from L1

**Metric proof to escalate:** L1 hit rate is high but p99 still misses target on cache miss path. Show L1 hit rate, miss rate, and latency breakdown for cache-miss requests.

**Rollback:** Feature flag to disable L1 (all reads go to Redis). L1 is purely additive — disabling it only affects latency, not correctness.

## Step 4: Add stampede protection (singleflight)

**When:** Cache miss causes thundering herd — many concurrent requests for the same expired key all hit the database simultaneously.

**Actions:**

- Add `singleflight.Group` per key in-process (deduplicates concurrent fetches within one instance)
- Optionally add distributed lock with short lease (e.g., 2s) for cross-instance deduplication
- First requester fetches from DB, others wait on the singleflight result
- Populate cache asynchronously but ensure first requester still meets p99

**Metric proof to escalate:** Singleflight resolves stampede, but CPU is now the bottleneck (not I/O). Show CPU profile flamegraph with marshal/unmarshal dominating.

**Rollback:** Feature flag to disable singleflight (requests go directly to cache/DB). Singleflight is purely additive.

## Step 5: Zero-serialization read path

**When:** Flamegraph shows >10% CPU spent on JSON marshal/unmarshal in the hot path. Response format matches storage format.

**Actions:**

- Pre-process data at write time or on pub/sub receive (parse once)
- Store raw bytes in L1 cache, serve directly to HTTP response
- Use callback hooks (e.g., `OnSetLocalCache`) to customize cache entry format per use case
- Avoid re-marshaling on every request

**Metric proof to escalate:** CPU is still the bottleneck despite zero-ser. Show flamegraph with remaining hot spots (lock contention, allocation pressure).

**Rollback:** Feature flag to fall back to standard marshal/unmarshal path.

**Reference implementation:** See `github.com/huykn/distributed-cache` examples/heavy-read-api — demonstrates OnSetLocalCache callback achieving 74K req/s with zero-copy reads (44% throughput gain over direct Redis).

## Step 6: Lock-free patterns and advanced concurrency

**When:** Mutex contention or allocation pressure appears in the profile. Throughput target is >10K req/s per instance.

**Actions:**

- Replace mutex-guarded counters with `atomic.AddInt64`
- Use `atomic.Value` for lock-free read/write of shared data
- Use `sync.Pool` for high-allocation hot paths (only when GC pressure is confirmed)
- Batch counter updates to reduce atomic contention at extreme throughput
- Use channel-based async processing to offload heavy work from hot path

**Metric proof:** This is the last step. If targets are still not met, revisit architecture (horizontal scaling, read replicas, CQRS).

**Rollback:** Each pattern should be behind a feature flag or isolated in its own module.

**Reference implementation:** See `github.com/huykn/distributed-cache` examples/heavy-write-api/poc — demonstrates lock-free atomic pop at 50K req/s per pod with MinuteStore pattern.

**Full case study:** See [Voucher Distribution System](case-study-voucher-system.md) for a real system that progressed through all six steps.

## Decision Summary

| Step | Trigger condition | Typical throughput range | Complexity added |
|---|---|---|---|
| 1. Fix queries | DB queries dominate profile | 0 - 2K RPS | Low |
| 2. Redis cache | Repeated reads, data tolerates staleness | 2K - 10K RPS | Low-Medium |
| 3. L1 cache | Redis round-trip is the bottleneck | 10K - 50K RPS | Medium |
| 4. Singleflight | Thundering herd on cache miss | 10K - 50K RPS | Low |
| 5. Zero-serialization | CPU bound at marshal/unmarshal | 50K - 100K+ RPS | Medium |
| 6. Lock-free patterns | Mutex contention in profile | 50K - 100K+ RPS | High |

**The RPS ranges are indicative, not prescriptive.** The actual trigger is always the profile, not the traffic number. A poorly written 500 RPS service may need step 2. A well-written 20K RPS service may not need step 5.
