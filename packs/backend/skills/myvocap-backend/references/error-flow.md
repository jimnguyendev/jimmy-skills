# Error Flow

The MyVocab backend uses a 4-stage error pipeline. Every error travels through exactly one path — no shortcuts, no dual logging.

## The Chain

```
Repository                    Service                     Handler                     httpresponse
   │                            │                           │                            │
   ├─ pgx.ErrNoRows            │                           │                            │
   │  → thingNotFound(id)      │                           │                            │
   │    (wraps sentinel)       │                           │                            │
   │                           │                           │                            │
   ├─ other DB error           │                           │                            │
   │  → fmt.Errorf(            │                           │                            │
   │      "feature: op: %w",   │                           │                            │
   │      err)                 │                           │                            │
   │                           │                           │                            │
   └──────────────────────────→│                           │                            │
                               ├─ ownership check          │                            │
                               │  → thingNotFound(id)      │                            │
                               │                           │                            │
                               ├─ quota check              │                            │
                               │  → ErrQuotaExceeded       │                            │
                               │                           │                            │
                               └──────────────────────────→│                            │
                                                           │                            │
                                            writeError(c, lgr, err)                     │
                                                           │                            │
                                                    toAppError(err)                     │
                                                           │                            │
                                                           ├─ errors.Is(ErrNotFound)    │
                                                           │  → httpresponse.NotFound() │
                                                           │                            │
                                                           ├─ errors.Is(ErrQuota...)    │
                                                           │  → httpresponse.Unproc()   │
                                                           │                            │
                                                           ├─ unmapped                  │
                                                           │  → pass through as-is      │
                                                           │                            │
                                                           └──────────────────────────→ │
                                                                                        │
                                                                              WriteError(c, lgr, err)
                                                                                        │
                                                                              ├─ *AppError? → write status
                                                                              │    └─ 5xx? → log cause
                                                                              │
                                                                              └─ bare error? → 500
                                                                                   └─ log full error
```

## Stage 1: Repository — Wrap with Context

```go
// Convert pgx.ErrNoRows → domain sentinel
func (r *postgresRepo) FindByID(ctx context.Context, id string) (Thing, error) {
    row, err := r.q.GetThing(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return Thing{}, thingNotFound(id) // wraps ErrThingNotFound
    }
    if err != nil {
        return Thing{}, fmt.Errorf("thing: find: %w", err) // wrap with feature:operation
    }
    return thingFromRow(row), nil
}
```

**Rules:**

- `pgx.ErrNoRows` → always convert to domain sentinel
- Other errors → wrap with `fmt.Errorf("<feature>: <operation>: %w", err)`
- Never return raw pgx/sql errors to service layer
- Never import `httpresponse` in repository

## Stage 2: Service — Semantic Validation + Sentinel Errors

```go
func (s *service) Get(ctx context.Context, userID, id string) (Thing, error) {
    item, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return Thing{}, err // sentinel passes through
    }
    if item.UserID != userID {
        return Thing{}, thingNotFound(id) // ownership → not-found (don't leak existence)
    }
    return item, nil
}

func (s *service) Create(ctx context.Context, userID string, in CreateInput) (Thing, error) {
    count, _ := s.repo.CountByUser(ctx, userID)
    if count >= MaxThingsPerUser {
        return Thing{}, fmt.Errorf("%w", ErrQuotaExceeded) // business rule → sentinel
    }
    return s.repo.Create(ctx, userID, in)
}
```

**Rules:**

- Service returns sentinel errors or repo errors (already wrapped)
- Ownership failures → `NotFound` (not `Forbidden`) to avoid leaking existence
- Quota/state violations → sentinel errors
- Never log AND return — pick one (return; let WriteError log 5xx)

## Stage 3: Handler — Single-Line Error Dispatch

```go
func (h *Handler) Get(c *gin.Context) {
    uid, ok := h.resolveUser(c)
    if !ok {
        return
    }
    item, err := h.svc.Get(c.Request.Context(), uid, c.Param("id"))
    if err != nil {
        writeError(c, h.lgr, err) // ONE line — no if-else chain
        return
    }
    httpresponse.Success(c, item)
}
```

**Two error paths in handlers:**

| Error type | How to handle |
|---|---|
| Syntactic (bad JSON, missing field) | `httpresponse.Error(c, http.StatusBadRequest, err)` |
| Domain (from service) | `writeError(c, h.lgr, err)` |

Never mix them. Never write `if errors.Is(...)` in a handler.

## Stage 4: httpresponse.WriteError — Terminal Boundary

```go
func WriteError(c *gin.Context, lgr errorLogger, err error) {
    if ae, ok := errors.AsType[*AppError](err); ok {
        // Known error → write its status
        if ae.Status >= 500 && lgr != nil {
            lgr.Error(ctx, ae.Message, "code", ae.Code, "err", ae.Cause)
        }
        c.AbortWithStatusJSON(ae.Status, envelope{...})
        return
    }
    // Unknown error → 500 + log
    lgr.Error(ctx, "unhandled error", "err", err)
    c.AbortWithStatusJSON(500, envelope{...})
}
```

**Rules:**

- This is the ONLY place 5xx errors are logged
- 4xx errors are NOT logged (they're expected client mistakes)
- Unknown errors become 500 automatically — no need to handle every case

## Sentinel Error Declaration Patterns

### Simple (one entity type)

```go
// In service.go or errors.go
var ErrThingNotFound = errors.New("thing not found")

// Wrapper adds context for logs
func thingNotFound(id string) error {
    return fmt.Errorf("%w: %s", ErrThingNotFound, id)
}
```

### Multiple entities in one feature

```go
// In errors.go
var (
    ErrDeckNotFound  = errors.New("deck not found")
    ErrCardNotFound  = errors.New("card not found")
    ErrQuotaExceeded = errors.New("quota exceeded")
)

func deckNotFound(id string) error {
    return fmt.Errorf("%w: %s", ErrDeckNotFound, id)
}

func cardNotFound(id string) error {
    return fmt.Errorf("%w: %s", ErrCardNotFound, id)
}
```

## Anti-Patterns (Blocked)

| Anti-pattern | Why it's wrong | Correct |
|---|---|---|
| `log.Error(...); return err` | Error logged twice (here + WriteError) | Just `return err` |
| `if errors.Is(err, ErrNotFound) { c.JSON(404, ...) }` in handler | Bypasses envelope; duplicates toAppError logic | `writeError(c, h.lgr, err)` |
| `httpresponse.NotFound(...)` in service | Service shouldn't know about HTTP | Return sentinel; map in `toAppError` |
| `return httpresponse.Internal("...", err)` in repo | Repo shouldn't know about HTTP | `return fmt.Errorf("feature: op: %w", err)` |
| Catch-all `default: return httpresponse.Internal(...)` in toAppError | Unmapped errors already become 500 via WriteError | Just `return err` |
