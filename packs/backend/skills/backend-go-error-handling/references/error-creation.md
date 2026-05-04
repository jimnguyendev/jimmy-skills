# Error Creation

## Errors as Values

Go treats errors as ordinary values implementing the `error` interface:

```go
type error interface {
    Error() string
}
```

This means errors are returned, not thrown. Every function that can fail returns an `error` as its last return value, and every caller must check it.

```go
// ✗ Bad — silently discarding errors
data, _ := os.ReadFile("config.yaml")

// ✗ Bad — only checking in some branches
result, err := doSomething()
fmt.Println(result) // using result without checking err

// ✓ Good — always check before using other return values
data, err := os.ReadFile("config.yaml")
if err != nil {
    return fmt.Errorf("reading config: %w", err)
}
```

## The Return Contract: nil error = usable value

A function returning `(T, error)` makes one promise: **if error is nil, the value is usable. If error is non-nil, the value is meaningless.**

```go
// ✓ Good — contract is clear
func GetUser(ctx context.Context, id string) (*User, error) {
    row := db.QueryRowContext(ctx, "SELECT ...", id)
    var u User
    if err := row.Scan(&u.ID, &u.Name); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound // not found = real error
        }
        return nil, fmt.Errorf("scanning user: %w", err)
    }
    return &u, nil // non-nil user, nil error — safe to use
}
```

### Never return nil, nil

Returning `nil` for both value and error breaks the contract. It forces every caller to add an extra nil check, and forgetting that check causes nil pointer panics:

```go
// ✗ Bad — nil, nil forces callers to double-check
func GetUser(ctx context.Context, id string) (*User, error) {
    row := db.QueryRowContext(ctx, "SELECT ...", id)
    var u User
    if err := row.Scan(&u.ID, &u.Name); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, nil // caller must now check both err AND user
        }
        return nil, err
    }
    return &u, nil
}

// Every caller becomes fragile:
user, err := GetUser(ctx, id)
if err != nil { ... }
if user == nil { ... } // easy to forget — nil pointer panic waiting to happen
```

### "Not found" is a real error

Treat "not found" as a sentinel error, not as a nil value. This matches `database/sql` using `sql.ErrNoRows`:

```go
var ErrUserNotFound = errors.New("user not found")

// Caller — one clear check
user, err := svc.GetUser(ctx, id)
if errors.Is(err, ErrUserNotFound) {
    // show "sign up" page or return 404
    return
}
if err != nil {
    // operation failed — maybe retry, maybe return 500
    return
}
// err is nil — user is guaranteed usable
fmt.Println(user.Name)
```

### Three-return alternative

If your team genuinely wants "not found is not an error", use a bool to signal presence:

```go
func GetUser(ctx context.Context, id string) (user *User, found bool, err error)
```

This is valid but less common in service methods. Most Go APIs keep it simple with `(nil, ErrUserNotFound)`. Whichever style you choose, **be consistent across the codebase** — mixing styles forces callers to remember which functions use which pattern.

## Error String Conventions

Error strings MUST be lowercase, without trailing punctuation, and should not duplicate the context that wrapping will add.

```go
// ✗ Bad — capitalized, punctuation, redundant prefix
return errors.New("Failed to connect to database.")
return fmt.Errorf("UserService: failed to fetch user: %w", err)

// ✓ Good — lowercase, no punctuation, concise
return errors.New("connection refused")
return fmt.Errorf("fetching user: %w", err)
```

When errors are wrapped through multiple layers, each layer adds its own prefix. The result reads like a chain:

```
creating order: charging card: connecting to payment gateway: connection refused
```

## Creating Errors

### `errors.New` — static error messages

```go
var ErrNotFound = errors.New("not found")
var ErrUnauthorized = errors.New("unauthorized")
```

### `fmt.Errorf` — dynamic error messages

```go
// ✗ Avoid — high-cardinality message, each user/tenant combo is a unique string
return fmt.Errorf("user %s not found in tenant %s", userID, tenantID)

// ✓ Prefer — static message here, structured attributes at the log site
return errors.New("user not found")
```

See [Low-Cardinality Error Messages](#low-cardinality-error-messages) for why this matters.

### Decision table: which error strategy to use

| Situation | Strategy | Example |
| --- | --- | --- |
| Caller needs to match a specific condition | Sentinel error (`errors.New` as package var) | `var ErrNotFound = errors.New("not found")` |
| Caller needs to extract structured data | Custom error type | `type ValidationError struct { Field, Msg string }` |
| Error is purely informational, not matched on | `fmt.Errorf` or `errors.New` | `fmt.Errorf("connecting to %s: %w", addr, err)` |
| Need structured data to travel with the error | Custom error type + `Unwrap()` | See [Structured Context Belongs in Logs](./error-handling.md#structured-context-belongs-in-logs) |

## Low-Cardinality Error Messages

APM and log aggregation tools (Datadog, Loki, Sentry) group errors by message. When you interpolate variable data into error strings, every unique combination creates a separate group — dashboards become unusable and alerting breaks.

```go
// ✗ Bad — high cardinality: each file/line combo creates a unique error message
fmt.Errorf("error in %s at line %d of the csv", csvPath, line)

// ✓ Good — static error, structured attributes at the log site
err := errors.New("csv parsing error")
// ... later, at the logging boundary:
h.logger.Error(ctx, "csv parsing failed",
    "err", err,
    "csv_file_path", csvPath,
    "csv_file_line", line,
)
```

If the variable data must travel up the stack, use a custom error type with `Unwrap()` and extract its fields once when logging or recording telemetry.

**Static wrapping prefixes are fine** — `fmt.Errorf("fetching user: %w", err)` is low-cardinality because the prefix never changes. What to avoid is interpolating IDs, paths, counts, or other variable data into the message itself.

## Custom Error Types

Create custom error types when callers need to extract structured data from errors.

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// Usage
func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    return nil
}
```

### Custom types that wrap other errors

Implement `Unwrap()` so `errors.Is` and `errors.As` can traverse the chain:

```go
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error {
    return e.Err
}
```
