# Caching Patterns

The fastest code is code that doesn't run. Caching pre-computed results, deduplicating concurrent requests, and avoiding unnecessary work are often the highest-leverage performance improvements.

## Compiled Pattern Caching

**Diagnose:** 1- `go tool pprof` (CPU profile) — look for `regexp.Compile`, `regexp.MustCompile`, or `template.Parse` appearing in hot paths; their presence means patterns are being recompiled per call instead of once 2- `go test -bench -benchmem` — benchmark per-call compilation vs cached version; expect 10-12x improvement and allocs/op dropping to zero for the compilation step

### Regexp at package level

`regexp.Compile` parses a pattern into a state machine — ~5,700ns per compilation. Match operations on a compiled regexp cost ~450ns. Compiling per-call wastes 10-12x:

```go
// Bad — compiled on every call
func isValid(email string) bool {
    re := regexp.MustCompile(`^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$`)
    return re.MatchString(email)
}

// Good — compiled once, safe for concurrent use
var emailRegex = regexp.MustCompile(`^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$`)

func isValid(email string) bool { return emailRegex.MatchString(email) }
```

Note: `regexp.MustCompile` panics on invalid patterns — fine for package-level constants (caught at startup). Use `regexp.Compile` for user-provided patterns. Go's regexp uses linear-time matching (no backtracking).

### Template caching

`template.Parse` is equally expensive. Parse once at startup:

```go
var reportTmpl = template.Must(template.ParseFiles("templates/report.html"))
```

### Precomputed lookup tables

When a computation is pure (same input → same output) and the input space is small, replace calculation with array lookup:

```go
var hexDigit = [16]byte{'0','1','2','3','4','5','6','7','8','9','a','b','c','d','e','f'}

func byteToHex(b byte) (byte, byte) {
    return hexDigit[b>>4], hexDigit[b&0x0f] // two array lookups vs branching logic
}
```

If the table fits in L1/L2 cache, lookup is faster than even simple computation.

## Request-Level Caching

**Diagnose:** 1- `go tool pprof` (goroutine profile) — look for many goroutines blocked on the same external call (HTTP fetch, DB query); this signals a cache stampede where N goroutines all miss the cache simultaneously 2- `fgprof` — shows off-CPU wait time; look for the same fetch function dominating wall-clock time across many goroutines, confirming duplicated concurrent work 3- `go tool pprof -alloc_objects` — check if cache miss handling allocates heavily; high alloc counts on fetch functions confirm the stampede is also generating GC pressure

### singleflight for cache stampede prevention

When a cache entry expires, many goroutines may simultaneously discover the miss and all request the same expensive computation. `singleflight` ensures only one goroutine fetches while others wait:

```go
import "golang.org/x/sync/singleflight"

var (
    cache sync.Map
    sf    singleflight.Group
)

func GetWeather(city string) (string, error) {
    if val, ok := cache.Load(city); ok {
        return val.(string), nil
    }

    // Only one goroutine fetches; others block on the same key
    result, err, _ := sf.Do(city, func() (any, error) {
        data, err := fetchFromAPI(city)
        if err == nil { cache.Store(city, data) }
        return data, err
    })
    return result.(string), err
}
```

→ See `jimmy-skills@backend-go-concurrency` skill for `singleflight` API details and `sync.Map` vs `RWMutex` decision guidance. → **Generics alternative:** Use `github.com/samber/go-singleflightx` to avoid interface{} boxing overhead; expect 2-4x faster result retrieval compared to the standard library's `singleflight.Group`.

### LRU caches

For bounded caches with eviction, the standard library's `container/list` works but has poor cache locality (each node is a separate heap allocation). For high-performance LRU:

- **`github.com/hashicorp/golang-lru`** — thread-safe, simple API
- **`github.com/elastic/go-freelru`** — merges hashmap and ringbuffer into contiguous memory, ~37x faster than sharded implementations

When using third-party cache libraries, refer to the library's official documentation for current API signatures.

## Algorithmic Complexity

**Diagnose:** 1- `go tool pprof` (CPU profile) — look for functions with high cumulative time that contain nested loops or repeated linear scans; these are algorithmic complexity bottlenecks 2- `go test -bench` — benchmark with different input sizes (100, 1K, 10K, 100K); if time grows quadratically (10x input → 100x time), the algorithm is O(n²) and needs replacement

Before micro-optimizing, check that the algorithm itself isn't the bottleneck. A constant-factor improvement on an O(n²) algorithm loses to a naive O(n log n) implementation at scale.

**Common complexity traps in Go:**

| Pattern | Complexity | Fix | Fixed complexity |
| --- | --- | --- | --- |
| `slices.Contains` in a loop | O(n·m) | Build `map[T]struct{}` first, then lookup | O(n+m) |
| Nested loops for matching | O(n²) | Index with a map, sort+binary search, or `slices.BinarySearch` | O(n log n) or O(n) |
| Repeated `append` without prealloc | O(n²) amortized copies | `make([]T, 0, n)` | O(n) |
| String concatenation with `+=` | O(n²) total copies | `strings.Builder` | O(n) |
| Linear scan for min/max/dedup | O(n) per query | Sort once, query many times | O(n log n) + O(log n) per query |

**Think in Big-O first, then optimize constants.** A 10x constant-factor improvement matters; switching from O(n²) to O(n) matters more.

## Work Avoidance

**Diagnose:** 1- `go tool pprof` (CPU profile) — look for linear scan functions (`slices.Contains`, `slices.Index`) or iterator chains (`Filter`, `Map`) consuming CPU in hot paths 2- `go test -bench` — benchmark the current approach vs a map-based or early-return version; expect O(n) → O(1) for membership tests, significant improvement for short-circuit loops

### Map lookups over slice scanning

`Contains(slice, element)` is O(n). Map lookups are O(1). When doing multiple membership tests against the same collection, build a map once:

```go
// Bad — O(n*m), checking Contains per element
for _, item := range subset {
    if !Contains(collection, item) { return false } // O(n) per check
}

// Good — O(n+m), build map once, O(1) lookups
seen := make(map[T]struct{}, len(collection))
for _, item := range collection { seen[item] = struct{}{} }
for _, item := range subset {
    if _, ok := seen[item]; !ok { return false }
}
```

Use `struct{}` (0 bytes) instead of `bool` (1 byte) for set maps.

### Early returns and short-circuit loops

Return immediately when the answer is known. Finding the target on iteration 3 of 1000 saves 997 iterations:

```go
// Bad — always iterates full collection
found := false
for _, item := range collection {
    if item == target { found = true }
}
return found

// Good — returns on first match
for i := range collection {
    if collection[i] == target { return true }
}
return false
```

### Avoid iterator chains

Chaining iterator operations (`Filter → Map → First`) creates closures and intermediate machinery. A direct loop is simpler and faster:

```go
// Bad — creates 2 iterators with closures
result, ok := First(Filter(collection, predicate))

// Good — single pass, early return, no closures
for i := range collection {
    if predicate(collection[i]) { return collection[i], true }
}
```

### Replace indirect function calls with direct loops

When a function wraps another function (e.g., `FromSlicePtr` calling `Map` with a closure), the closure indirection prevents inlining. Replace with a direct loop:

```go
// Bad — Map() with closure, per-element function call overhead
func FromSlicePtr(items []*T) []T {
    return Map(items, func(p *T) T { return *p })
}

// Good — direct loop, inlineable, -13% to -17% time
func FromSlicePtr(items []*T) []T {
    result := make([]T, len(items))
    for i := range items { result[i] = *items[i] }
    return result
}
```

## Two-Level Cache (L1 In-Process + L2 Redis)

**Diagnose:** 1- Distributed tracing or `fgprof` — if Redis round-trip (typically 1-3ms) dominates p99 despite high cache hit rate, an in-process L1 eliminates network hops for hot keys 2- `go tool pprof` (goroutine profile) — many goroutines blocked on Redis calls signals that Redis is the bottleneck, not the computation

When a single Redis cache layer is not enough because the network round-trip itself is the bottleneck, add an in-process L1 cache in front of Redis.

```
Read path:
  L1 (in-process, ~100ns) → HIT → return
  L1 MISS → singleflight.Do(key) → L2 (Redis, ~1-3ms) → populate L1 → return

Write path:
  Set L1 → serialize → Set L2 → publish invalidation via pub/sub
  Other pods: receive event → update or delete from their L1
```

### Implementation pattern

```go
type SyncedCache struct {
    local   LocalCache          // L1: in-process (LRU or LFU)
    store   RemoteStore         // L2: Redis
    sync    Synchronizer        // Redis Pub/Sub for cross-pod invalidation
    sfGroup singleflight.Group  // Stampede protection on cache miss
}

func (sc *SyncedCache) Get(ctx context.Context, key string) (any, bool) {
    // L1 check — ~100ns, no network
    if val, ok := sc.local.Get(key); ok {
        return val, true
    }

    // L2 check — deduplicated via singleflight
    result, _, _ := sc.sfGroup.Do(key, func() (any, error) {
        // Double-check L1 (another goroutine may have populated it)
        if val, ok := sc.local.Get(key); ok {
            return val, nil
        }
        data, err := sc.store.Get(ctx, key)
        if err != nil {
            return nil, nil
        }
        var val any
        json.Unmarshal(data, &val)
        sc.local.Set(key, val, 1) // populate L1
        return val, nil
    })
    return result, result != nil
}
```

### Stale write-back prevention

In read-heavy architectures with separate reader and writer pods, reader pods should NOT write back to Redis from L1. A reader may hold stale L1 data and overwrite fresher data in Redis.

```go
type Options struct {
    // When false (default), reader pods skip Redis writes.
    // Only writer pods should write to Redis.
    ReaderCanSetToRedis bool
}
```

### Cross-pod invalidation

Use Redis Pub/Sub to notify other pods when data changes. Two modes:

| Mode | Payload | Other pods action | Best for |
|---|---|---|---|
| Set (propagate) | Full serialized value | Store in L1 directly | Small values, read-heavy |
| Invalidate | Key only | Delete from L1, lazy-load on next read | Large values |

Self-invalidation prevention: each event carries a `SenderPodID`. Pods ignore events they sent themselves.

### When to use two-level cache

- Redis hit rate is already >90% but Redis round-trip dominates p99
- Read-heavy workload with predictable hot keys
- Multiple pods serve the same data

### When NOT to use

- Single-instance deployment (no cross-pod sync needed — just use in-process cache)
- Write-heavy workload (invalidation storms negate L1 benefit)
- Data changes every request (L1 cache will always be stale)

→ See `jimmy-skills@engineering-perf-optimization-process` for the escalation ladder that determines when to add L1 cache.

## Zero-Serialization Read Path

**Diagnose:** `go tool pprof` (CPU profile) — if `json.Marshal` or `json.Unmarshal` appears in the top 10 functions on the hot path, consuming >10% of CPU, zero-serialization eliminates that cost.

The traditional cache read path marshals and unmarshals on every request:

```
Redis → []byte → Unmarshal → Object → Marshal → []byte → HTTP Response
```

When the response format matches the storage format (JSON in, JSON out), skip the round-trip:

```
L1 Cache → []byte → HTTP Response (direct write)
```

### Implementation pattern

Pre-process data when it arrives via pub/sub, not on every read:

```go
// CachedPost stores pre-processed data for zero-cost reads
type CachedPost struct {
    Hash string   // Pre-extracted for ETag/HTTP 304
    Data []byte   // Raw JSON bytes — serve directly to HTTP response
}

// OnSetLocalCache callback: parse ONCE when data arrives, not on every read
cfg.OnSetLocalCache = func(event InvalidationEvent) any {
    post, err := parsePost(event.Value)
    if err != nil {
        return nil
    }
    return &CachedPost{
        Hash: post.Hash,
        Data: event.Value, // Keep raw bytes for direct response
    }
}
```

```go
// Handler: zero-cost read — no marshal, no unmarshal, no allocation
func handleGetPost(w http.ResponseWriter, r *http.Request) {
    val, found := cache.Get(ctx, postID)
    if !found {
        http.Error(w, "not found", 404)
        return
    }
    cached := val.(*CachedPost)
    w.Header().Set("Content-Type", "application/json")
    w.Write(cached.Data) // Direct byte write
}
```

### Benchmark evidence

On Apple M3 Pro (11 cores), 4 threads, 400 connections, 10s duration:

| Endpoint | Req/sec | P50 | P99 |
|---|---|---|---|
| Zero-serialization (L1 → raw bytes) | **74,720** | **4.98ms** | **12.50ms** |
| L1 + re-marshal every request | 71,891 | 5.16ms | 14.40ms |
| Direct Redis (no L1) | 51,807 | 7.01ms | 27.26ms |

44% throughput gain over Redis direct. 2.1x lower P99 latency.

### When to use

- Read-heavy API (>>writes)
- Response format matches storage format
- CPU profile shows marshal/unmarshal in hot path (>10% CPU)
- Hot data served from L1 cache

### When NOT to use

- Response format differs from storage format (transformation needed anyway)
- Write-heavy API (data changes frequently, pre-processing waste)
- CPU is not the bottleneck (I/O or DB is)
