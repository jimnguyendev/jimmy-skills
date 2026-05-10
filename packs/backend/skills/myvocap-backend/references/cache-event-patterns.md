# Cache-Aside & Event Bus Patterns

## Cache Layer (`internal/cache/`)

The cache package provides typed, generic Redis operations with JSON serialization and singleflight deduplication.

### Core API

```go
// Create cache instance with key prefix
c := cache.New(rdb, "myvocap:")

// Single key operations
cache.Set(ctx, c, "user:1", user, time.Hour)
u, err := cache.Get[User](ctx, c, "user:1")              // (zero, nil) on miss
err := cache.Delete(ctx, c, "user:1")                     // invalidate
cache.Forget(c, "user:1")                                 // clear singleflight entry

// Batch operations
cache.MSetWithTTL(ctx, c, items, time.Hour)                // pipeline write
```

### Cache-Aside: FetchOne

Single-entity cache-aside with automatic write-through on miss:

```go
const (
    cacheKeyPrefix = "thing:"
    cacheTTL       = 15 * time.Minute
)

func cacheKey(id string) string { return cacheKeyPrefix + id }

func (s *service) Get(ctx context.Context, id string) (Thing, error) {
    if s.cache != nil {
        return cache.FetchOne(ctx, s.cache, cacheKey(id), func(ctx context.Context) (Thing, error) {
            return s.repo.FindByID(ctx, id)
        }, cacheTTL)
    }
    return s.repo.FindByID(ctx, id)
}
```

Flow: check cache → miss → call fetcher → store result → return.

### Cache-Aside: FetchWithSingleflight

Batch cache-aside with singleflight deduplication (prevents thundering herd):

```go
func (s *service) ListByIDs(ctx context.Context, ids []string) ([]Thing, error) {
    if s.cache == nil {
        return s.repo.ListByIDs(ctx, ids)
    }
    return cache.FetchWithSingleflight(
        ctx, s.cache, ids,
        cacheKey,                                          // func(id string) string
        func(t Thing) string { return t.ID },              // extract ID from entity
        "thing:list-by-ids",                               // singleflight group key
        s.repo.ListByIDs,                                  // batch fetcher
        cacheTTL,
    )
}
```

Flow: build keys → MGet → unmarshal hits → collect miss IDs → singleflight-guarded DB fetch → MSet misses → return all.

### Cache Invalidation on Write

Always pair `Delete` + `Forget` on writes:

```go
func (s *service) Update(ctx context.Context, id string, in UpdateInput) (Thing, error) {
    item, err := s.repo.Update(ctx, id, in)
    if err != nil {
        return Thing{}, err
    }

    // Invalidate cache entry + singleflight in-flight request
    if s.cache != nil {
        if err := cache.Delete(ctx, s.cache, cacheKey(id)); err != nil {
            s.lgr.Warn(ctx, "cache delete failed", "key", cacheKey(id), "err", err)
        }
        cache.Forget(s.cache, cacheKey(id))
    }

    return item, nil
}
```

**Rules:**

- `cache.Delete` removes the Redis key
- `cache.Forget` clears the singleflight entry (so the next read actually hits DB)
- Both are best-effort — log warning on failure, don't fail the operation
- Always do BOTH on any write (create/update/delete)

### Graceful Nil Degradation

Cache is optional. When `s.cache == nil`, skip caching entirely:

```go
func (s *service) Get(ctx context.Context, id string) (Thing, error) {
    if s.cache != nil {
        return cache.FetchOne(...)
    }
    return s.repo.FindByID(ctx, id) // fallback to DB-only
}
```

This pattern is used in every service. The `Deps.Cache` field may be nil.

---

## Event Bus (`internal/event/`)

In-process pub/sub for eventual consistency without external broker overhead.

### Core Types

```go
// Publisher is the interface services depend on (not *Bus directly)
type Publisher interface {
    Publish(ctx context.Context, e Event) error      // sync: handlers run in-request
    PublishAsync(ctx context.Context, e Event) error  // async: handlers in goroutines
}

// Noop returns a Publisher that discards all events
func Noop() Publisher

// Event is the payload
type Event struct {
    ID      string // auto-generated UUID
    Topic   string
    Payload any
}

func NewEvent(topic string, payload any) Event
```

### Topic Constants

Each feature declares its topics in `events.go`:

```go
package thing

const (
    TopicThingCreated = "thing:created"
    TopicThingUpdated = "thing:updated"
    TopicThingDeleted = "thing:deleted"
)
```

Format: `<feature>:<action>`, lowercase, colon-separated.

### Publishing Events

```go
// In service — always async, always best-effort
func (s *service) Create(ctx context.Context, in CreateInput) (Thing, error) {
    item, err := s.repo.Create(ctx, in)
    if err != nil {
        return Thing{}, err
    }

    // Publish is non-fatal: the entity is already persisted
    if err := s.eventBus.PublishAsync(ctx, event.NewEvent(TopicThingCreated, item)); err != nil {
        s.lgr.Warn(ctx, "publish event failed", "topic", TopicThingCreated, "err", err)
    }
    return item, nil
}
```

**Rules:**

- Use `PublishAsync` (not `Publish`) — handlers shouldn't block the request
- Log warning on publish failure, don't fail the operation
- Publish AFTER the write succeeds, not before
- Payload is the domain entity (or its ID for deletes)

### Subscribing to Events

Listeners live in `listeners.go`, called from `Provide()`:

```go
func registerListeners(bus *event.Bus, lgr logger.Logger) {
    if bus == nil {
        return
    }

    bus.Subscribe(TopicThingCreated, "thing.on-created", func(ctx context.Context, e event.Event) error {
        created := e.Payload.(Thing) // type-assert — bus guarantees the type
        lgr.Info(ctx, "thing created", "id", created.ID)
        return nil
    })
}
```

**Rules:**

- Guard `bus == nil` — bus is optional
- Subscriber ID format: `<feature>.<handler-name>`
- Type-assert payload directly — the publisher controls the type
- Return error to log it (bus catches and logs); don't panic
- Handlers are panic-safe — one panic doesn't affect others

### Wiring in Provide()

```go
func Provide(d Deps) *Handler {
    // Degrade to Noop if no bus configured
    eb := event.Noop()
    if d.EventBus != nil {
        eb = d.EventBus
    }

    repo := NewPostgresRepository(d.Postgres)
    svc := NewService(repo, d.Cache, eb, d.Lgr)  // service gets Publisher interface
    registerListeners(d.EventBus, d.Lgr)          // listeners get *Bus (for Subscribe)
    return NewHandler(svc, d.Lgr)
}
```

Note: service receives `event.Publisher` (interface), listeners receive `*event.Bus` (concrete, for `Subscribe`).

### Bus Lifecycle

```go
// In cmd/app/http.go
eventBus := event.New(lgr)

// ... wire features ...

cleanup := func(_ context.Context) error {
    eventBus.Close() // FIRST: drain async handlers before closing deps
    rdb.Close()
    dbConn.Close()
    return nil
}
```

`eventBus.Close()` must be called BEFORE closing Redis/DB — async handlers may still be using them.

### Cross-Feature Events

Feature A publishes → Feature B subscribes. The listener registration happens in `cmd/app/http.go`, not inside either feature:

```go
// cmd/app/http.go
eventBus.Subscribe("thing:created", "analytics.on-thing-created", func(ctx context.Context, e event.Event) error {
    thing := e.Payload.(thingpkg.Thing)
    return analyticsSvc.TrackCreation(ctx, thing.ID)
})
```

This keeps features decoupled — neither imports the other.
