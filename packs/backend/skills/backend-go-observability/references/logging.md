# Structured Logging Fundamentals

> **Team standard:** Use `prep-go-log` for all Go services. See [prep-go-log reference](./prep-go-log.md) for initialization, dependency injection, and usage patterns. This document covers the universal principles that apply regardless of logging library.

→ See `jimmy-skills@backend-go-error-handling` skill for the single handling rule.

## Why Structured Logging

Structured logs emit key-value pairs instead of freeform strings. Log management systems (Datadog, Grafana Loki, CloudWatch) can index, filter, and aggregate structured fields — something impossible with `log.Printf` output.

```go
// ✗ Bad — freeform string, impossible to filter by user_id
log.Printf("ERROR: failed to create user %s: %v", userID, err)

// ✓ Good — structured key-value pairs, machine-parseable
logger.Error(ctx, "user creation failed",
    "user_id", userID,
    "err", err,
)
// JSON output: {"time":"2025-01-15T10:30:00Z","level":"ERROR","msg":"user creation failed","user_id":"u-123","err":"connection refused","chain_id":"req-abc-123"}
```

## Logger Setup

With `prep-go-log`, the handler setup is handled by the library. See [prep-go-log reference](./prep-go-log.md) for initialization.

- **Production:** `prepzap.NewLogger(ctx, attr.Production, svcName, signozExporter, prepzap.WithAsync())` — JSON output, async batch processing, OTel export to Signoz
- **Development:** `prepzap.NewLogger(ctx, attr.Local, svcName, signozExporter)` — human-readable output
- **Testing:** `&console.Logger{Env: "test"}` — simple stdout, no dependencies

## Log Levels

```go
logger.Debug(ctx, "cache lookup", "key", cacheKey, "hit", false)
logger.Info(ctx, "order created", "order_id", orderID, "total", amount)
logger.Warn(ctx, "rate limit approaching", "current_usage", 0.92, "limit", 1000)
logger.Error(ctx, "payment failed", "order_id", orderID, "err", err)
```

**Rule of thumb**: if you're unsure between Warn and Error, ask "did the operation succeed?" If yes (even with degradation), use Warn. If no, use Error.

## Cost of Logging

Logging is not free. Each log line costs CPU (serialization), I/O (disk/network), and money (log ingestion/storage in your aggregation platform). The cost scales with volume, which is directly controlled by log level.

- **Debug level in production** can generate millions of log lines per minute in a busy service, overwhelming your log pipeline and inflating costs by 10-100x
- **Info level** is the typical production default — it provides enough visibility without excessive volume
- Debug level SHOULD be disabled in production — use Info level in production and Debug only in development or when actively debugging a specific issue
- For high-throughput services, consider lowering verbosity or adding sampling in your log pipeline so verbose logs do not overwhelm ingestion and storage

## Logging with Context

Every `prep-go-log` method requires `ctx` as the first argument. This propagates chain log ID and trace context automatically — no manual `*Context` variant needed.

```go
// ✗ Bad — context.Background() loses chain log ID and trace context
logger.Error(context.Background(), "query failed", "err", err)

// ✓ Good — request context carries chain_id, trace_id, span_id
logger.Error(ctx, "query failed", "err", err)
```

## Request-Scoped Attributes via Chain Log ID

`prep-go-log` uses chain log ID middleware instead of child loggers. The middleware injects a unique ID into context at the request boundary, and every log call automatically includes it:

```go
// HTTP: middleware injects chain_id
c.SetContextValue(attr.ChainLogIdKey, id)

// gRPC: interceptor injects chain_id
ctxWithVal := context.WithValue(ctx, attr.ChainLogIdKey, chainID)

// All downstream log calls automatically include chain_id
logger.Info(ctx, "order processed", "order_id", orderID)
// Output: {"chain_id":"req-abc-123", "msg":"order processed", "order_id":"o-456"}
```

→ See [prep-go-log reference](./prep-go-log.md) for full middleware setup (HTTP, gRPC, Kafka).

## Migrating to prep-go-log

If a service currently uses Zap, Logrus, zerolog, or slog directly, migrate to `prep-go-log`. It provides unified OTel integration, Signoz export, and chain log ID propagation out of the box.

**Step 1:** Add `prep-go-log` and initialize in `cmd/*/main.go`

**Step 2:** Replace all direct logger calls with the `log.Logger` interface:

```go
// zap → prep-go-log
// Before: zap.L().Info("order created", zap.String("order_id", id))
// After:
logger.Info(ctx, "order created", "order_id", id)

// logrus → prep-go-log
// Before: logrus.WithField("order_id", id).Info("order created")
// After:
logger.Info(ctx, "order created", "order_id", id)

// slog → prep-go-log
// Before: slog.InfoContext(ctx, "order created", "order_id", id)
// After:
logger.Info(ctx, "order created", "order_id", id)
```

**Step 3:** Add chain log ID middleware and remove the old logger dependency.

→ See [prep-go-log reference](./prep-go-log.md) for full initialization and DI patterns.

## Common Logging Mistakes

```go
// ✗ Bad — errors MUST be either logged OR returned, NEVER both (single handling rule violation)
if err != nil {
    s.logger.Error(ctx, "query failed", "err", err)
    return fmt.Errorf("query: %w", err) // error gets logged twice up the chain
}

// ✓ Good — return with context, log at the top level
if err != nil {
    return fmt.Errorf("querying users: %w", err)
}

// ✗ Bad — NEVER log PII (emails, SSNs, passwords, tokens)
logger.Info(ctx, "user logged in", "email", user.Email, "ssn", user.SSN)

// ✓ Good — log identifiers, not sensitive data
logger.Info(ctx, "user logged in", "user_id", user.ID)
```
