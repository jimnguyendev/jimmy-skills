# Cross-feature Kafka publisher wiring

How analytics, notification, and other cross-feature Kafka publishers get built, injected, and torn down. Source: ADR-0021. Reference implementation: `cmd/app/analytics_publishers.go`.

## Why a separate file

`cmd/app/http.go` is the wiring file for the HTTP process. Its job is to construct infrastructure clients and inject adapters into feature `Deps`. Constructors that fit the "one infra client used by many features" shape (`postgres.NewConn`, `dbredis.NewClient`, `event.New`) are one-line expressions called inline.

Analytics-style publishers do NOT fit that shape:

- There is no single Kafka client used everywhere — `pkg/kafka` builds **one producer per topic**, each typed by a sample event.
- Each producer is wrapped by an adapter that lives in the feature **owning the consumer-side interface** (`gamification.NewKafkaReviewPublisher`, `analytics.NewKafkaPracticePublisher`, ...).
- Failure mode is **best-effort**: a dead broker disables analytics but never blocks user write paths. Each producer's init must be isolated.
- `Close` requires per-producer nil-checks plus structured error logs.

Inline wiring drowns `http.go`'s real responsibility (Gin, middleware, routes) and forces it to import `pkg/kafka`, `internal/events`, `internal/analytics`, `internal/feature/gamification`. Adding a new event family meant editing `http.go` in three places. The sibling-file pattern fixes all three problems.

## File location and naming

```
cmd/app/
  http.go                       <-- Gin / middleware / route wiring
  consumer.go                   <-- Kafka consumer process
  analytics_publishers.go       <-- THIS pattern
  notification_publishers.go    <-- next family follows the same shape
```

One file per area. Area name = the concern the publishers serve in aggregate (analytics, notification, audit), NOT a feature name. If two features happen to publish into the same area's topics, both publishers live in the same `_publishers.go` file.

## Required shape

Every `<area>_publishers.go` has exactly three pieces.

### 1. The struct

Holds publisher adapters (typed by the consumer-side interface from the owning feature) plus the underlying `kafka.Producer` instances retained for `Close`.

```go
type analyticsPublishers struct {
    reviewPub   vocabulary.ReviewEventPublisher // may be nil
    practicePub review.PracticeEventPublisher   // may be nil

    // Underlying producers retained so Close can release them. Nil when the
    // matching publisher above is nil.
    reviewProducer   kafka.Producer
    practiceProducer kafka.Producer
}
```

Notes:

- Publisher field type is the **feature's consumer-side interface**, not the concrete adapter. Feature `Deps` declares the same interface — wiring just hands it the implementation.
- Producer fields are the raw `kafka.Producer` so `Close` can call them. Without these, `Close` would have to know the adapter's internal struct.
- Every publisher field is documented as "may be nil". This is the contract — the consuming feature MUST be nil-safe.

### 2. The builder

```go
func buildAnalyticsPublishers(ctx context.Context, cfg config.Config, lgr logger.Logger) *analyticsPublishers {
    pubs := &analyticsPublishers{}

    if len(cfg.Kafka.Brokers) == 0 {
        lgr.Info(ctx, "kafka: no brokers configured, analytics publishers disabled")
        return pubs
    }

    if p, err := kafka.NewProducer(&events.ReviewSubmittedEvent{}, cfg.Kafka.Brokers); err == nil {
        pubs.reviewProducer = p
        pubs.reviewPub = gamification.NewKafkaReviewPublisher(p, lgr)
    } else {
        lgr.Warn(ctx, "kafka: review producer init failed, analytics review events disabled", "err", err)
    }

    if p, err := kafka.NewProducer(&events.PracticeAttemptEvent{}, cfg.Kafka.Brokers); err == nil {
        pubs.practiceProducer = p
        pubs.practicePub = analytics.NewKafkaPracticePublisher(p, lgr)
    } else {
        lgr.Warn(ctx, "kafka: practice producer init failed, analytics practice events disabled", "err", err)
    }

    return pubs
}
```

Required behaviour:

- **Per-producer init isolation.** One topic failing to spin up MUST NOT break the others. Each producer init has its own `if err == nil { ... } else { Warn }` branch.
- **Empty-broker short-circuit.** If `cfg.Kafka.Brokers` is empty, log Info and return the zero struct. Local dev without Kafka should be a one-line decision, not a startup failure.
- **Always returns a non-nil struct**, even when nothing succeeded. Callers never check `pubs == nil`; they only check individual fields if they care.
- **Never propagate errors.** The signature is `*analyticsPublishers`, not `(*analyticsPublishers, error)` — analytics liveness must not gate process startup.

Adding a new publisher: build the producer, build the adapter, attach both. Keep the failure-mode shape (Warn + nil publisher) so the rest of the codebase can stay nil-safe.

### 3. The closer

```go
func (a *analyticsPublishers) Close(lgr logger.Logger) {
    if a.reviewProducer != nil {
        if err := a.reviewProducer.Close(); err != nil {
            lgr.Warn(context.Background(), "kafka: review producer close failed", "err", err)
        }
    }
    if a.practiceProducer != nil {
        if err := a.practiceProducer.Close(); err != nil {
            lgr.Warn(context.Background(), "kafka: practice producer close failed", "err", err)
        }
    }
}
```

Required behaviour:

- **Nil-skipping.** Every producer field is nil-checked.
- **Errors logged, never propagated.** Shutdown ordering MUST NOT fail because Kafka close was slow.
- **Idempotent.** Safe to call from both `closeStartupDeps` (early-error) and `cleanup` (normal shutdown). `kgo.Client.Close` is a no-op after the first call.

## How `http.go` consumes it

```go
// --- Analytics publishers (best-effort Kafka, per ADR-0009) ---
analyticsPubs := buildAnalyticsPublishers(ctx, cfg, lgr)

closeStartupDeps := func() {
    eventBus.Close()
    analyticsPubs.Close(lgr)
    _ = rdb.Close()
    dbConn.Close()
}

// ... feature wiring ...
vocabHandler := vocabulary.Provide(vocabulary.Deps{
    // ... other deps ...
    ReviewPub: analyticsPubs.reviewPub, // may be nil
})
```

`http.go` only:

1. Calls the builder once.
2. Adds `Close` to the cleanup chain.
3. References the field on each consuming feature's `Deps`.

It never imports `pkg/kafka`, `internal/events`, or anything publisher-internal.

## Consuming feature contract

The feature whose `Deps` carries the publisher MUST be nil-safe. Pattern:

```go
// In the service:
func (s *service) submitReview(ctx context.Context, in ReviewInput) error {
    // ... write the row ...

    if s.reviewPub == nil {
        return nil
    }
    if err := s.reviewPub.Publish(ctx, ReviewSubmittedEvent{...}); err != nil {
        s.lgr.Warn(ctx, "publish review event failed", "err", err)
        // do NOT return err — analytics is a mirror, not a source of truth
    }
    return nil
}
```

Notes:

- **Nil publisher = no-op.** Local dev without Kafka must keep working.
- **Publish failure logs and continues.** Per ADR-0009 analytics is best-effort.
- The publisher interface itself lives in the feature package (consumer-side). The wiring file imports the feature package to pick up the constructor for the adapter — that import is intentional and pushes the cross-feature surface into `cmd/app/`, where it belongs.

## Adding a new publisher

1. **Owning feature** declares the consumer-side interface (`type FooEventPublisher interface { Publish(ctx, FooEvent) error }`) in its package.
2. **Owning feature** ships an adapter (`func NewKafkaFooPublisher(p kafka.Producer, lgr logger.Logger) FooEventPublisher`).
3. **`<area>_publishers.go`** gets one new struct field, one new builder branch (matching the failure-mode shape), one new close branch.
4. **`http.go`** gets one extra line in the consuming feature's `Deps` injection. No new imports.

Total touch on `http.go`: one line per `Deps` injection. No imports.

## When to create a new `<area>_publishers.go` file

Add a new file per **distinct area** the publishers serve in aggregate. Examples:

- `analytics_publishers.go` — review, practice, login, streak, diamond events that feed ClickHouse.
- `notification_publishers.go` — push / email events that feed the notification consumer.
- `audit_publishers.go` — security / compliance events that feed the audit log.

Do NOT split per feature ("`gamification_publishers.go`") — the area name reflects the consumer side, not the producer side. If two features publish into the same area, both adapters live in the same file.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Inline `kafka.NewProducer(...)` in `http.go` | Move to `cmd/app/<area>_publishers.go` |
| Builder returns `(*publishers, error)` and aborts process startup on failure | Return `*publishers` only; log Warn + nil field per failed producer |
| Builder builds all producers in one `if err != nil { return err }` chain | Each producer needs its own `if err == nil { ... } else { Warn }` branch |
| Consuming feature panics / errors when publisher field is nil | Nil-check first; nil = no-op (analytics is best-effort) |
| Consuming feature returns publish errors to the caller | Log Warn + continue; analytics never blocks the user write path |
| Publisher adapter constructor moved to a central `internal/publishers/` package | Constructor stays in the owning feature (consumer-side interface rule, ADR-0009) |
| Splitting per producer feature (`gamification_publishers.go`) | Split per consumer area (`analytics_publishers.go`) |
| Forgetting to add `Close` to BOTH `closeStartupDeps` and `cleanup` | Both paths must call it; `Close` is idempotent so calling twice is safe |
