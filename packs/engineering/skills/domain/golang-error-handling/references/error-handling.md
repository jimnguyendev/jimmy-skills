# Error Handling Patterns and Logging

## The Single Handling Rule

An error MUST be handled exactly once: either log it or return it, never both. Doing both causes duplicate log entries and makes debugging harder.

```go
// ✗ Bad — logs AND returns (duplicate noise)
func processOrder(id string) error {
    err := chargeCard(id)
    if err != nil {
        log.Printf("failed to charge card: %v", err)
        return fmt.Errorf("charging card: %w", err)
    }
    return nil
}

// ✓ Good — return with context, let the caller decide
func processOrder(id string) error {
    err := chargeCard(id)
    if err != nil {
        return fmt.Errorf("charging card: %w", err)
    }
    return nil
}

// ✓ Good — handle at the top level (HTTP handler, main, etc.)
func handleOrder(w http.ResponseWriter, r *http.Request) {
    err := processOrder(r.FormValue("id"))
    if err != nil {
        logger.Error("order failed",
            zap.Error(err),
            zap.String("order_id", r.FormValue("id")),
        )
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

## Panic and Recover

### When to panic

Panic MUST only be used for truly unrecoverable states — programmer errors, impossible conditions, or corrupt invariants. NEVER use panic for expected failures like network timeouts or missing files.

```go
// ✓ Acceptable — programmer error in initialization
func MustCompileRegex(pattern string) *regexp.Regexp {
    re, err := regexp.Compile(pattern)
    if err != nil {
        panic(fmt.Sprintf("invalid regex %q: %v", pattern, err))
    }
    return re
}

// ✗ Bad — panic for a normal failure
func GetUser(id string) *User {
    user, err := db.Find(id)
    if err != nil {
        panic(err) // callers cannot recover gracefully
    }
    return user
}
```

### Recovering from panics

Use `recover` in deferred functions at goroutine boundaries (HTTP handlers, worker goroutines) to prevent one panic from crashing the entire process.

```go
func safeHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if r := recover(); r != nil {
                logger.Error("panic recovered",
                    zap.Any("panic", r),
                    zap.ByteString("stack", debug.Stack()),
                )
                http.Error(w, "internal error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

If you need stack traces in production, capture them at recovery boundaries with `debug.Stack()` or rely on your error reporting backend to collect them once at the top level.

## Structured Context Belongs in Logs

Keep returned errors focused on the failure itself. Attach high-cardinality context such as request IDs, user IDs, tenant IDs, or HTTP metadata at the logging/tracing boundary where you already know the execution context.

When the context must travel with the error across layers, prefer a small custom error type over ad-hoc formatting:

```go
type OrderError struct {
    OrderID string
    Err     error
}

func (e *OrderError) Error() string { return "order operation failed" }

func (e *OrderError) Unwrap() error { return e.Err }

func createOrder(ctx context.Context, orderID string) error {
    if err := insertOrder(ctx, orderID); err != nil {
        return &OrderError{OrderID: orderID, Err: fmt.Errorf("inserting order: %w", err)}
    }
    return nil
}
```

At the boundary, extract the structured fields once and log them with `zap`:

```go
if err := createOrder(ctx, orderID); err != nil {
    var orderErr *OrderError
    if errors.As(err, &orderErr) {
        logger.Error("create order failed", zap.Error(err), zap.String("order_id", orderErr.OrderID))
    } else {
        logger.Error("create order failed", zap.Error(err))
    }
}
```

## Logging Errors with `zap`

→ See `jimmy-skills@golang-observability` skill for comprehensive structured logging guidance, including logger setup, log levels, log handlers, HTTP middleware, and cost considerations.
