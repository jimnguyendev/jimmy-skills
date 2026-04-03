# prep-go-log — Internal Logging Library

`prep-go-log` is the team's internal structured logging library. It wraps Zap (or Logrus) behind a unified `log.Logger` interface with built-in OpenTelemetry integration and Signoz export. All Go services MUST use this library instead of calling Zap, Logrus, or slog directly.

**Module:** `gitlab.testsprep.online/packages/prep-go-log`

## Logger Interface — Local Copy Pattern

Each project defines its **own** `log.Logger` interface that mirrors prep-go-log's interface. This keeps the prep-go-log import boundary tight — only `cmd/*/main.go` and middleware import prep-go-log directly; all other code imports the project's local `pkg/log`.

```go
// YOUR_PROJECT/pkg/log/logger.go — local interface, NOT imported from prep-go-log
package log

import "context"

type Logger interface {
    Info(ctx context.Context, msg string, args ...interface{})
    Debug(ctx context.Context, msg string, args ...interface{})
    Warn(ctx context.Context, msg string, args ...interface{})
    Error(ctx context.Context, msg string, args ...interface{})
    Fatal(ctx context.Context, msg string, args ...interface{})
    Panic(ctx context.Context, msg string, args ...interface{})
}
```

**Why a local copy?** The project depends on an interface it owns, not on prep-go-log's type directly. This means:

- Swapping logger backends requires changes **only** in `cmd/*/main.go`
- All internal packages import `your-project/pkg/log`, never `prep-go-log/pkg/log`
- `prepzap.NewLogger()` satisfies the local interface because the method signatures match

**Import boundary rules:**

| Package | Imports from prep-go-log | Why |
| --- | --- | --- |
| `cmd/*/main.go` | `prepzap`, `signoz`, `attr` | Initialization only |
| `internal/infra/mdw/` | `attr` (for `ChainLogIdKey`) | Middleware injects chain ID |
| `internal/infra/interceptor/` | `attr` (for `ChainLogIdKey`) | gRPC interceptor injects chain ID |
| `internal/infra/kafka/` | `attr` (for `ChainLogIdKey`) | Consumer injects chain ID |
| **Everything else** | **Nothing** — imports `pkg/log` local | Clean dependency boundary |

## Initialization

Initialize once in `cmd/*/main.go`, then inject everywhere via constructors.

### Production (Zap + Signoz)

From `cmd/http/main.go` in learning-quest-platform:

```go
import (
    "your-project/pkg/log"           // local interface — used everywhere
    "prep-go-log/pkg/attr"           // only in cmd/ and middleware
    "prep-go-log/pkg/export/signoz"  // only in cmd/
    "prep-go-log/pkg/log/prepzap"    // only in cmd/
)

func main() {
    ctx := context.Background()
    mode := app.Str2Mode(os.Getenv("MODE"))

    // 1. Create Signoz exporter (gRPC for performance)
    signoz := signoz.NewExporter(_svcname, config.App.OtlpLoggerAddr, signoz.WithEnableGRPC())

    // 2. Create logger — WithAsync() is the standard production option
    logger := prepzap.NewLogger(ctx, attr.Txt2Env(mode.String()), _svcname, signoz, prepzap.WithAsync())

    // 3. Set up distributed tracing and register cleanup on shutdown
    cleanup := signoz.Trace(ctx)

    // 4. Inject logger into ALL constructors — it satisfies the local log.Logger interface
    gamifiDB, _ := database.NewLearningGamifi(ctx, logger, config)
    userRepo := repository.NewUserCache(lnMysqlDB, lnMongoDB, lpDB, logger, rdb)
    questRepo := repository.NewQuestCached(gamifiDB, logger, rdb)
    dgsvc := service.NewDailygoal(logger, config, questRepo, dgRepo, userRepo, ...)
    streakSvc := streak.NewService(logger, config, userRepo, streakRepo, questRepo)
    dgHandler := handler.NewDailyGoal(responder, dgsvc)
    streakHandler := handler.NewStreak(mode, responder, streakSvc)

    // 5. Register shutdown — close logger and trace exporter
    app.RegisterShutdownFn(
        rdb.Close,
        gamifiDB.Close,
        func(_ context.Context) error { return cleanup() },
    )
}
```

**Additional prepzap options** (use only when tuning is needed):

```go
prepzap.WithBatchSize(50)                 // flush every N logs (default: 50)
prepzap.WithBatchTimeout(10*time.Second)  // or every N seconds (default: 10s)
prepzap.WithAsyncBufferSize(2048)         // queue capacity (default: 2048)
```

### Testing / Local Development (Console)

```go
import "gitlab.testsprep.online/packages/prep-go-log/pkg/log/console"

logger := &console.Logger{Env: "test"}
// Prints to stdout: [INFO] test: message
// No Signoz dependency, no network calls
```

## Dependency Injection Pattern

The logger is injected as the project's **local** `log.Logger` interface through constructors at every layer. Never use a global logger. Never import prep-go-log outside of `cmd/` and middleware.

```go
import "your-project/pkg/log"  // local interface, NOT prep-go-log

// internal/application/service/dailygoal.go
type Dailygoal struct {
    logger       log.Logger
    config       *config.Config
    questRepo    repository.Quest
    dgRepo       repository.DailyGoal
    userRepo     repository.User
    qpool        *quest.PoolGenerator
    lvlPredictor level.Predictor
    idGenerator  identity.Generator
}

func NewDailygoal(
    logger log.Logger,
    config *config.Config,
    questRepo repository.Quest,
    dgRepo repository.DailyGoal,
    userRepo repository.User,
    qpool *quest.PoolGenerator,
    lvlPredictor level.Predictor,
    idGenerator identity.Generator,
) *Dailygoal {
    return &Dailygoal{
        logger:       logger,
        config:       config,
        questRepo:    questRepo,
        // ...
    }
}
```

The same pattern applies to **all layers** (from learning-quest-platform):

| Layer | Real example | Import |
| --- | --- | --- |
| Service | `service.NewDailygoal(logger, config, questRepo, dgRepo, userRepo, ...)` | `pkg/log` |
| Repository | `repository.NewStreakBitmap(logger, rdb)` | `pkg/log` |
| Domain Builder | `l0x.NewL0XBuilder(logger, questRepo, userRepo, ...)` | `pkg/log` |
| Topic Handler | `topichdl.NewSyncDailyGoal(config, logger, dgRepo, questRepo, ...)` | `pkg/log` |
| Infrastructure | `database.NewLearningGamifi(ctx, logger, config)` | `pkg/log` |
| Presentation | `handler.NewStreak(mode, responder, streakSvc)` | (logger via responder) |

## Structured Fields

Pass key-value pairs as variadic `...interface{}` arguments after the message:

```go
// Single field
logger.Error(ctx, "failed to set streak bit", "err", err)

// Multiple fields
logger.Warn(ctx, "failed to predict user level, using default",
    "user_id", userID,
    "line_id", lineID,
    "err", err,
)

// Complex values
logger.Info(ctx, "quest build completed",
    "user_id", userID,
    "line_id", lineID,
    "quest_count", len(quests),
    "duration_ms", elapsed.Milliseconds(),
)
```

**Field naming conventions:**

- Use `snake_case` for field keys: `"user_id"`, `"line_id"`, `"quest_count"`
- Always include `"err", err` when logging errors
- Do NOT interpolate values into the message string — use structured fields instead

```go
// ❌ Bad — high-cardinality message, not filterable
logger.Info(ctx, fmt.Sprintf("User %d booked homestay %s", userID, homestayID))

// ✅ Good — static message, structured fields for filtering
logger.Info(ctx, "homestay booked", "user_id", userID, "homestay_id", homestayID)
```

## Chain Log ID (Request Tracing)

Every request gets a unique chain log ID injected into context via middleware. The logger automatically extracts it from `ctx` and attaches it to every log line, enabling end-to-end request tracing across services.

### HTTP Middleware (Gin)

```go
// internal/infra/mdw/request_logger.go
type RequestLogger struct {
    idGenerator identity.Generator
}

func (l *RequestLogger) AddLogChainIdentify() websv.MiddlewareFn {
    return func(c websv.Context) error {
        id, err := l.idGenerator.Gen()
        if err != nil {
            return nil
        }
        c.SetContextValue(attr.ChainLogIdKey, id)
        return nil
    }
}
```

### gRPC Interceptor

```go
// internal/infra/interceptor/rpc_logger.go
func (l *RPCLogger) AddUnaryLogChainIdentify() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        chainID, _ := l.idGenerator.Gen()
        ctxWithVal := context.WithValue(ctx, attr.ChainLogIdKey, chainID)
        return handler(ctxWithVal, req)
    }
}
```

### Kafka Consumer

```go
// internal/infra/kafka/consumer.go — each message gets its own chain ID
id, err := c.idgen.Gen()
if err != nil {
    id = ""
}
ctxWithVal := context.WithValue(ctx, attr.ChainLogIdKey, id)

if err := handle(ctxWithVal, message); err != nil {
    c.logger.Error(ctxWithVal, "error processing message", "topic", topic, "err", err)
}
```

The chain log ID is automatically included in every log entry when using `prep-go-log`, enabling you to filter all logs from a single request across all services in Signoz.

## Available Implementations

| Implementation | Package | Backend | Use case |
| --- | --- | --- | --- |
| **prepzap** (recommended) | `pkg/log/prepzap` | Zap SugaredLogger | Production services |
| **preplogrus** | `pkg/log/preplogrus` | Logrus | Legacy services migrating to prepzap |
| **console** | `pkg/log/console` | fmt.Println | Tests, local development |

All three implement `log.Logger`, so switching backends requires no code changes outside `cmd/*/main.go`.

## Environment Modes

```go
import "gitlab.testsprep.online/packages/prep-go-log/pkg/attr"

attr.Local      // "local" — development machine
attr.Develop    // "develop" — dev environment
attr.Staging    // "staging" — pre-production
attr.Testing    // "testing" — test environment
attr.Production // "production" — live
```

Mode affects log format (JSON in production, text in local) and is attached as a structured field to every log entry.

## OpenTelemetry Integration

prep-go-log automatically bridges with OpenTelemetry:

- **Zap backend:** Uses `otelzap.NewCore()` to send logs to the OTel LoggerProvider
- **Logrus backend:** Uses `otellogrus.NewHook()` to add OTel hook

Logs exported to Signoz include trace context, enabling correlation between logs and traces in the Signoz UI.

### Distributed Tracing Setup

```go
// In cmd/*/main.go, after creating the Signoz exporter:
cleanup := signozExpt.Trace(ctx)
defer func() {
    if err := cleanup(); err != nil {
        logger.Error(ctx, "trace cleanup failed", "err", err)
    }
}()
```

## Common Mistakes

These anti-patterns were found in real production code (learning-quest-platform):

```go
// ❌ Bad — fmt.Sprintf in message (breaks log aggregation, high cardinality)
// Found in: internal/infra/kafka/consumer.go, internal/domain/quest/l0x/builder.go
logger.Error(ctx, fmt.Sprintf("[%s] Error processing message: %v. Err: %v", topic, string(msg.Value), err))

// ✅ Good — static message, structured fields for filtering
logger.Error(ctx, "error processing message", "topic", topic, "err", err)
```

```go
// ❌ Bad — inconsistent field key naming (mixing camelCase and snake_case)
// Found in: streak code uses "userId", service code uses "user_id"
s.logger.Error(ctx, "failed to set streak bit", "userId", userId)
dg.logger.Warn(ctx, "failed to predict level", "userID", userID)

// ✅ Good — always use snake_case for field keys
s.logger.Error(ctx, "failed to set streak bit", "user_id", userID)
```

```go
// ❌ Bad — log AND return (single handling rule violation)
if err != nil {
    s.logger.Error(ctx, "failed to charge card", "err", err)
    return fmt.Errorf("charging card: %w", err)
}

// ✅ Good — return only, let the caller handle logging
if err != nil {
    return fmt.Errorf("charging card: %w", err)
}
```

```go
// ❌ Bad — importing prep-go-log in internal/ service or domain code
import "gitlab.testsprep.online/packages/prep-go-log/pkg/log"

// ✅ Good — import the project's local interface
import "your-project/pkg/log"
```

```go
// ❌ Bad — not passing request context (chain log ID and trace context lost)
logger.Info(context.Background(), "order created", "order_id", id)

// ✅ Good — pass the request context
logger.Info(ctx, "order created", "order_id", id)
```
