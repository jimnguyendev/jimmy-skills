# Structured Logging with `slog`

→ See `jimmy-skills@backend-go-error-handling` skill for the single handling rule.

## Why Structured Logging

Structured logs emit key-value pairs instead of freeform strings. Log management systems (Datadog, Grafana Loki, CloudWatch) can index, filter, and aggregate structured fields — something impossible with `log.Printf` output.

```go
// ✗ Bad — freeform string, impossible to filter by user_id
log.Printf("ERROR: failed to create user %s: %v", userID, err)

// ✓ Good — structured key-value pairs, machine-parseable
slog.Error("user creation failed",
    "user_id", userID,
    "error", err,
)
// JSON output: {"time":"2025-01-15T10:30:00Z","level":"ERROR","msg":"user creation failed","user_id":"u-123","error":"connection refused"}
```

## Handler Setup

```go
// Production MUST use JSON — because plain-text multiline logs (eg. stack traces) would be split into separate records by log collectors
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// Development — human-readable text
logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))

slog.SetDefault(logger)
```

## Log Levels

```go
slog.Debug("cache lookup", "key", cacheKey, "hit", false)
slog.Info("order created", "order_id", orderID, "total", amount)
slog.Warn("rate limit approaching", "current_usage", 0.92, "limit", 1000)
slog.Error("payment failed", "order_id", orderID, "error", err)
```

**Rule of thumb**: if you're unsure between Warn and Error, ask "did the operation succeed?" If yes (even with degradation), use Warn. If no, use Error.

## Cost of Logging

Logging is not free. Each log line costs CPU (serialization), I/O (disk/network), and money (log ingestion/storage in your aggregation platform). The cost scales with volume, which is directly controlled by log level.

- **Debug level in production** can generate millions of log lines per minute in a busy service, overwhelming your log pipeline and inflating costs by 10-100x
- **Info level** is the typical production default — it provides enough visibility without excessive volume
- Debug level SHOULD be disabled in production — use `slog.LevelInfo` in production and `slog.LevelDebug` only in development or when actively debugging a specific issue
- For high-throughput services, consider lowering verbosity or adding sampling in your log pipeline so verbose logs do not overwhelm ingestion and storage

## Logging with Context

MUST use the `*Context` variants to correlate logs with the current trace. When an OpenTelemetry bridge is configured, trace_id and span_id are automatically injected into log records.

```go
// ✗ Bad — no trace correlation
slog.Error("query failed", "error", err)

// ✓ Good — trace_id/span_id attached automatically when OTel bridge is active
slog.ErrorContext(ctx, "query failed", "error", err)
```

## Adding Request-Scoped Attributes

Use `slog.With()` to create a child logger that includes attributes on every log line. Middleware can inject request-scoped fields so all downstream logs carry the same context.

```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        logger := slog.With(
            "request_id", r.Header.Get("X-Request-ID"),
            "method", r.Method,
            "path", r.URL.Path,
        )
        // Store enriched logger in context for downstream use
        ctx := WithLogger(r.Context(), logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Log Sinks and the `slog` Ecosystem

`slog` supports pluggable handlers. Start simple:

- `slog.JSONHandler` — JSON to stdout/stderr for production
- `slog.TextHandler` — human-readable key=value logs for development
- [lmittmann/tint](https://github.com/lmittmann/tint) — colorized terminal output for local use

Prefer shipping JSON logs to stdout and let your platform collector (Datadog Agent, Fluent Bit, Vector, OpenTelemetry Collector, CloudWatch agent) route them to the final backend. This keeps application code and log transport concerns separate.

For additional handlers and middleware, use the ecosystem catalog at [go.dev/wiki/Resources-for-slog](https://go.dev/wiki/Resources-for-slog) and choose libraries that match your framework and deployment constraints.

## Migrating from zap / logrus / zerolog

`log/slog` is the standard library logger since Go 1.21. If the project uses `zap`, `logrus`, or `zerolog`, migrate to `slog` — it has a stable API, broad ecosystem support, and eliminates an unnecessary dependency.

**Step 1: Stabilize output** — configure `slog` to emit the same field names and JSON shape you need in production so old and new call sites can coexist temporarily without breaking dashboards or parsers.

**Step 2: Replace call sites** — change all logger calls to `slog`:

```go
// zap → slog
// Before: zap.L().Info("order created", zap.String("order_id", id))
// After:
slog.Info("order created", "order_id", id)

// logrus → slog
// Before: logrus.WithField("order_id", id).Info("order created")
// After:
slog.Info("order created", "order_id", id)

// zerolog → slog
// Before: log.Info().Str("order_id", id).Msg("order created")
// After:
slog.Info("order created", "order_id", id)
```

**Step 3: Remove the bridge** — once all call sites are migrated, replace the bridge handler with a native `slog` handler and remove the old logger dependency:

```go
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
})))
```

## Common Logging Mistakes

```go
// ✗ Bad — errors MUST be either logged OR returned, NEVER both (single handling rule violation)
if err != nil {
    slog.Error("query failed", "error", err)
    return fmt.Errorf("query: %w", err) // error gets logged twice up the chain
}

// ✓ Good — return with context, log at the top level
if err != nil {
    return fmt.Errorf("querying users: %w", err)
}

// ✗ Bad — NEVER log PII (emails, SSNs, passwords, tokens)
slog.Info("user logged in", "email", user.Email, "ssn", user.SSN)

// ✓ Good — log identifiers, not sensitive data
slog.Info("user logged in", "user_id", user.ID)
```
