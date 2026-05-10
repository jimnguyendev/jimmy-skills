---
name: myvocap-backend
description: "Project-specific backend patterns for MyVocab Go API. Enforces the exact feature module layout, Deps/Provide DI, sentinel→AppError error flow, sqlc-only repository, cache-aside with singleflight, and response envelope used in this codebase. MUST be loaded alongside generic backend-go-* skills — this skill overrides generic advice with repo-specific conventions."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents working on the MyVocab Go backend (server/).
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(sqlc:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are the MyVocab backend engineer. You know every pattern in this codebase intimately. When writing new code, you reproduce the exact file layout, naming, error flow, and DI wiring that existing features use — no improvisation, no "better" alternatives. When reviewing code, you check against the repo's pre-commit checklist.

**Precedence:** This skill overrides generic `jimmy-skills@backend-go-*` skills when they conflict. The codebase conventions documented here are the source of truth.

**Modes:**

- **Build mode** — adding a new feature or endpoint: scaffold from the feature module template, wire in `cmd/app/http.go`, run quality gates.
- **Review mode** — auditing code changes: run the pre-commit checklist, flag violations.

## Quick Reference

| Concept | Pattern | Reference |
|---|---|---|
| Feature layout | `internal/feature/<name>/{types,repository,service,handler,routes,provider,errors}.go` | `references/feature-module-template.md` |
| DI wiring | `Deps` struct + `Provide(d Deps) *Handler` | `references/feature-module-template.md` |
| Error flow | sentinel → `toAppError()` → `writeError()` → `httpresponse.WriteError()` | `references/error-flow.md` |
| Repository | sqlc-only (`sqlcgen.Queries`), domain type conversion via `*FromRow()` | `references/feature-module-template.md` |
| Caching | `cache.FetchOne` / `cache.FetchWithSingleflight` + graceful nil degradation | `references/cache-event-patterns.md` |
| Events | `event.Bus` / `event.Publisher` + `Noop()` fallback + `registerListeners()` | `references/cache-event-patterns.md` |
| Response | `httpresponse.Success/Paginated/Error/WriteError` envelope (all success = 200) | `references/response-envelope.md` |
| Middleware | auth → localUser → rateLimit chain; consumer-side interfaces | `references/middleware-wiring.md` |
| User context | `middleware.LocalUserIDFromCtx(ctx)` — never call user service from handler | `references/middleware-wiring.md` |
| Validation | handler = syntactic (`ShouldBindJSON`), service = semantic (state/quota/ownership) | below |
| Pagination | `ginx.QueryPage(c, maxSize)` → `db.Page` → repo uses `.Offset()` | below |
| Transactions | `r.pool.Begin(ctx)` → `r.q.WithTx(tx)` → `defer r.rollback(ctx, tx)` → `tx.Commit(ctx)` | `references/feature-module-template.md` |

## Rules (Non-Negotiable)

### 1. Feature layout is fixed

Every feature lives in `internal/feature/<name>/` with exactly these files:

| File | Responsibility |
|---|---|
| `types.go` | Domain models + input/output structs. JSON tags `snake_case`. Binding tags for validation. |
| `repository.go` | `Repository` interface + `postgresRepo` implementation via `sqlcgen.Queries`. |
| `service.go` | `Service` interface + `service` struct. Business logic, cache, events. Sentinel errors declared here or in `errors.go`. |
| `handler.go` | `Handler` struct with `svc` + `lgr`. HTTP handlers. Syntactic validation only. |
| `routes.go` | `RegisterRoutes(rg *gin.RouterGroup, h *Handler)`. Route registration. |
| `provider.go` | `Deps` struct + `Provide(d Deps) *Handler`. Wiring: repo → service → handler. |
| `errors.go` | Sentinel errors, `errorLogger` interface, `writeError()`, `toAppError()`. |
| `events.go` | (optional) Topic constants: `Topic<Feature><Action> = "<feature>:<action>"`. |
| `listeners.go` | (optional) `registerListeners(bus *event.Bus, lgr logger.Logger)`. Called from `Provide()`. |

Do NOT invent parallel layouts, split into sub-packages, or merge files.

### 2. No init(), no mutable globals

Exceptions: compiled regex, `var _ Iface = (*T)(nil)` assertions, sentinel errors, topic constants.

### 3. Deps struct lists only what the feature needs

```go
type Deps struct {
    Postgres  *pgxpool.Pool
    Cache     *cache.Cache      // may be nil → service degrades
    EventBus  *event.Bus        // may be nil → Noop() publisher
    Lgr       logger.Logger
}
```

When a field becomes unused, delete it. Don't keep "for later".

### 4. Validation: two layers, no duplication

- **Handler**: `c.ShouldBindJSON(&in)` → 400 via `httpresponse.Error(c, http.StatusBadRequest, err)`
- **Service**: entity state, ownership, quotas, cross-entity rules → sentinel error → 4xx via `writeError`
- **No CHECK constraints / domain UNIQUEs in migrations.** DB is dumb persistence.

### 5. Error handling: wrap, map, handle once

- Wrap with context: `fmt.Errorf("<feature>: <operation>: %w", err)`
- Sentinel in `errors.go`: `var ErrNotFound = errors.New("<thing> not found")`
- Wrap sentinel with ID: `func thingNotFound(id string) error { return fmt.Errorf("%w: %s", ErrNotFound, id) }`
- Map in `toAppError()`: sentinel → `httpresponse.NotFound/BadRequest/Unprocessable/Conflict`
- Unmapped errors fall through to 500 automatically
- Log OR return, never both. `WriteError` logs 5xx.

### 6. Repository: sqlc-only

- All queries in `internal/db/queries/<feature>.sql`
- Generated code in `internal/db/sqlc/`
- Repository wraps `sqlcgen.Queries`: `r.q.MethodName(ctx, params)`
- Convert rows via `<thing>FromRow(row sqlcgen.Type) DomainType`
- `pgx.ErrNoRows` → domain sentinel (e.g., `thingNotFound(id)`)
- **Exception**: writable CTEs that sqlc can't parse → raw `pool.Query` with one-line comment
- **Exception**: crawl schema → raw SQL (schema churns, JSONB upserts)

### 7. Every external call has a bounded context

```go
ctx, cancel := context.WithTimeout(c.Request.Context(), 10*time.Second)
defer cancel()
```

### 8. User context from middleware, not handler

```go
uid, ok := middleware.LocalUserIDFromCtx(c.Request.Context())
if !ok {
    httpresponse.WriteError(c, h.lgr,
        httpresponse.Unauthorized("MISSING_USER_CONTEXT", "missing user context", nil))
    return
}
```

Extract into `resolveUser(c *gin.Context) (string, bool)` helper on the handler.

### 9. Cache graceful degradation

```go
if s.cache == nil {
    return s.repo.ListByIDs(ctx, ids)
}
return cache.FetchWithSingleflight(ctx, s.cache, ids, ...)
```

Invalidate on write: `cache.Delete()` + `cache.Forget()` (singleflight).

### 10. Event bus graceful degradation

```go
// In Provide():
eb := event.Noop()
if d.EventBus != nil {
    eb = d.EventBus
}

// In service:
if err := s.eventBus.PublishAsync(ctx, event.NewEvent(TopicCreated, entity)); err != nil {
    s.lgr.Warn(ctx, "publish event failed", "topic", TopicCreated, "err", err)
}
```

### 11. Consumer-side interfaces for cross-cutting

Middleware declares narrow interface → feature's service satisfies it → wired in `cmd/app/http.go`.

```go
// In middleware package:
type LocalUserResolver interface {
    LocalUserID(ctx context.Context, prepID int64, token string) (string, error)
}

// In cmd/app/http.go:
middleware.RequireLocalUser(userBundle.Svc, lgr)
```

### 12. Response helpers

| Action | Helper |
|---|---|
| Success (including POST create) | `httpresponse.Success(c, data)` |
| Paginated list | `httpresponse.Paginated(c, items, httpresponse.Pagination{Total, Page, PerPage})` |
| Syntactic error | `httpresponse.Error(c, http.StatusBadRequest, err)` |
| Domain error | `writeError(c, h.lgr, err)` (feature-local, maps via `toAppError`) |

All successful responses return 200. Per `jimmy-skills@engineering-rest-api-design`, POST returns 200 not 201.

### 13. Pagination

```go
// Handler:
page := ginx.QueryPage(c, 100) // max 100

// Service → Repo:
items, total, err := s.repo.List(ctx, page)

// Repo:
r.q.ListThings(ctx, sqlcgen.ListThingsParams{
    Limit:  int32(page.Size),
    Offset: int32(page.Offset()),
})
```

### 14. Route registration

```go
func RegisterRoutes(rg *gin.RouterGroup, h *Handler) {
    v1 := rg.Group("/api/v1")
    v1.Handle(http.MethodGet, "/<feature>", h.List)
    v1.Handle(http.MethodPost, "/<feature>", h.Create)
    v1.Handle(http.MethodGet, "/<feature>/:id", h.Get)
    v1.Handle(http.MethodPut, "/<feature>/:id", h.Update)
    v1.Handle(http.MethodDelete, "/<feature>/:id", h.Delete)
}
```

Use `http.Method*` constants. Use `v1.Handle()`, not `v1.GET()` / `v1.POST()`.

### 15. Wiring in cmd/app/http.go

```go
// 1. Create handler via Provide
thingHandler := thing.Provide(thing.Deps{
    Postgres: dbConn,
    Cache:    appCache,
    EventBus: eventBus,
    Lgr:      lgr,
})

// 2. Register routes on the correct group
thing.RegisterRoutes(localUser, thingHandler) // if needs user UUID
thing.RegisterRoutes(protected, thingHandler) // if no user UUID needed
```

## Pre-Commit Checklist

Before reporting any Go change as done, self-review:

1. **Layer**: handler=syntactic, service=semantic, repo=persistence, middleware=cross-cutting?
2. **Imports**: no feature importing another feature? middleware using consumer-side interface?
3. **`init()` / globals**: none introduced?
4. **Context**: every external call has `context.WithTimeout`?
5. **Errors**: wrapped with `%w`? handled once? sentinels mapped via `writeError`?
6. **Validation**: syntactic in handler, semantic in service, no duplication?
7. **Resources**: `defer Close()` right after acquire? cleanup order preserved?
8. **Unused**: dead fields/params deleted?
9. **Dead abstractions**: no interface with one impl "for mocking" unless test uses it?
10. **Quality gates**:

    ```bash
    cd server && go vet ./... && golangci-lint run && go test ./... -count=1
    ```

## Common Mistakes (Blocked)

| Mistake | Fix |
|---|---|
| Handler calling another feature's service | Use middleware consumer-side interface or event bus |
| Business logic in handler | Move to service; handler only does `ShouldBindJSON` + call service |
| `sql.NullString` in domain types | Use `*string` in domain; convert in `*FromRow()` |
| CHECK constraint in migration | Remove; validate in service layer |
| Logging error then returning it | Pick one. `WriteError` logs 5xx automatically |
| `cache.Delete` without `cache.Forget` | Always pair both on write operations |
| `gin.Context` passed to service | Pass `c.Request.Context()` instead |
| Route on wrong group | `protected` = no user UUID; `localUser` = has user UUID |
| Missing `defer cancel()` after `context.WithTimeout` | Always defer immediately |
| `r.q.WithTx(tx)` without `defer r.rollback(ctx, tx)` | Always defer rollback |

## Cross-References

- Architecture & DI: `jimmy-skills@backend-go-design-patterns`
- Error handling theory: `jimmy-skills@backend-go-error-handling`
- Context propagation: `jimmy-skills@backend-go-context`
- Database patterns: `jimmy-skills@backend-go-database`
- Testing: `jimmy-skills@backend-go-testing`
- API design: `jimmy-skills@engineering-rest-api-design`
