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
func (h *Handler) handleOrder(w http.ResponseWriter, r *http.Request) {
    err := h.service.processOrder(r.Context(), r.FormValue("id"))
    if err != nil {
        h.logger.Error(r.Context(), "order failed",
            "err", err,
            "order_id", r.FormValue("id"),
        )
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

## Error Translation Across Layers

Translate low-level errors into domain terms at the boundary where they cross layers. The caller should not need to know whether storage is PostgreSQL, Redis, or an HTTP service — but it does need to know whether the item was not found vs the operation failed.

```go
// internal/user/repository.go — translates sql errors to domain errors
var ErrUserNotFound = errors.New("user not found")

func (r *Repository) GetByID(ctx context.Context, id string) (*User, error) {
    row := r.db.QueryRowContext(ctx, "SELECT id, name, email FROM users WHERE id = $1", id)
    var u User
    if err := row.Scan(&u.ID, &u.Name, &u.Email); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound // translate to domain term
        }
        return nil, fmt.Errorf("querying user: %w", err) // preserve real failures
    }
    return &u, nil
}
```

### What to translate, what to preserve

| Low-level error | Domain translation | Why |
| --- | --- | --- |
| `sql.ErrNoRows` | `ErrUserNotFound` | Caller decides 404 vs "sign up" — doesn't need to know it was SQL |
| Connection refused, timeout | Wrap with context, do NOT hide | Caller needs to know the operation failed and might be retryable |
| Constraint violation (unique) | `ErrDuplicateEmail` or similar | Caller can show "email already taken" |
| Unknown/unexpected DB error | Wrap with `fmt.Errorf("querying user: %w", err)` | Preserve for debugging, do not mask |

```go
// internal/user/handler.go — caller uses domain errors, not sql errors
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.service.GetUser(r.Context(), chi.URLParam(r, "id"))
    if errors.Is(err, ErrUserNotFound) {
        http.Error(w, "user not found", http.StatusNotFound)
        return
    }
    if err != nil {
        h.logger.Error(r.Context(), "get user failed", "err", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    // err is nil — user is safe to use
    respondJSON(w, user)
}
```

**Do NOT** hide real failures behind vague domain errors. `"connection refused"` wrapped as `ErrUserNotFound` means the caller thinks the user doesn't exist when the database is actually down.

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
func (h *Handler) safeHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rv := recover(); rv != nil {
                h.logger.Error(r.Context(), "panic recovered",
                    "panic", rv,
                    "stack", string(debug.Stack()),
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

At the boundary, extract the structured fields once and log them with `prep-go-log`:

```go
if err := createOrder(ctx, orderID); err != nil {
    var orderErr *OrderError
    if errors.As(err, &orderErr) {
        h.logger.Error(ctx, "create order failed", "err", err, "order_id", orderErr.OrderID)
    } else {
        h.logger.Error(ctx, "create order failed", "err", err)
    }
}
```

## Logging Errors with `prep-go-log`

→ See `jimmy-skills@backend-go-observability` skill for comprehensive structured logging guidance, including `prep-go-log` setup, dependency injection, chain log ID middleware, and Signoz integration.
