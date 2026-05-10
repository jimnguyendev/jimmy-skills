# Feature Module Template

Copy-paste scaffolding for a new feature. Replace `<feature>` with the feature name (e.g., `bookmark`), `<Feature>` with PascalCase (e.g., `Bookmark`), and `<FEATURE>` with SCREAMING_SNAKE (e.g., `BOOKMARK`).

## File: types.go

```go
package <feature>

import "time"

// <Feature> is the domain model mapping to the <feature>s table.
type <Feature> struct {
	ID        string    `json:"id"`
	UserID    string    `json:"user_id"`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

// Create<Feature>Input is the request body for creating a <feature>.
type Create<Feature>Input struct {
	Name string `json:"name" binding:"required,min=1,max=255"`
}

// Update<Feature>Input is the request body for updating a <feature>.
type Update<Feature>Input struct {
	Name string `json:"name" binding:"required,min=1,max=255"`
}
```

**Rules:**

- JSON tags: `snake_case`
- Binding tags on input structs for syntactic validation
- Domain model has NO binding tags
- Use `*string`, `*int`, `*time.Time` for nullable fields — never `sql.NullString`

---

## File: repository.go

```go
package <feature>

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/jimnguyendev/myvocap/server/internal/db"
	sqlcgen "github.com/jimnguyendev/myvocap/server/internal/db/sqlc"
	"github.com/jimnguyendev/myvocap/server/pkg/identity"
)

// Repository defines the data-access contract for <feature>s.
type Repository interface {
	List(ctx context.Context, userID string, page db.Page) ([]<Feature>, int, error)
	FindByID(ctx context.Context, id string) (<Feature>, error)
	Create(ctx context.Context, userID string, in Create<Feature>Input) (<Feature>, error)
	Update(ctx context.Context, id string, in Update<Feature>Input) (<Feature>, error)
	Delete(ctx context.Context, id string) error
}

type postgresRepo struct {
	q *sqlcgen.Queries
}

func NewPostgresRepository(pool *pgxpool.Pool) Repository {
	return &postgresRepo{q: sqlcgen.New(pool)}
}

func (r *postgresRepo) List(ctx context.Context, userID string, page db.Page) ([]<Feature>, int, error) {
	total, err := r.q.Count<Feature>s(ctx, userID)
	if err != nil {
		return nil, 0, fmt.Errorf("<feature>: count: %w", err)
	}
	rows, err := r.q.List<Feature>s(ctx, sqlcgen.List<Feature>sParams{
		UserID: userID,
		Limit:  int32(page.Size),
		Offset: int32(page.Offset()),
	})
	if err != nil {
		return nil, 0, fmt.Errorf("<feature>: list: %w", err)
	}
	out := make([]<Feature>, len(rows))
	for i, row := range rows {
		out[i] = <feature>FromRow(row)
	}
	return out, int(total), nil
}

func (r *postgresRepo) FindByID(ctx context.Context, id string) (<Feature>, error) {
	row, err := r.q.Get<Feature>(ctx, id)
	if errors.Is(err, pgx.ErrNoRows) {
		return <Feature>{}, <feature>NotFound(id)
	}
	if err != nil {
		return <Feature>{}, fmt.Errorf("<feature>: find: %w", err)
	}
	return <feature>FromRow(row), nil
}

func (r *postgresRepo) Create(ctx context.Context, userID string, in Create<Feature>Input) (<Feature>, error) {
	row, err := r.q.Create<Feature>(ctx, sqlcgen.Create<Feature>Params{
		ID:     identity.NewID(),
		UserID: userID,
		Name:   in.Name,
	})
	if err != nil {
		return <Feature>{}, fmt.Errorf("<feature>: create: %w", err)
	}
	return <feature>FromRow(row), nil
}

func (r *postgresRepo) Update(ctx context.Context, id string, in Update<Feature>Input) (<Feature>, error) {
	row, err := r.q.Update<Feature>(ctx, sqlcgen.Update<Feature>Params{
		ID:   id,
		Name: in.Name,
	})
	if errors.Is(err, pgx.ErrNoRows) {
		return <Feature>{}, <feature>NotFound(id)
	}
	if err != nil {
		return <Feature>{}, fmt.Errorf("<feature>: update: %w", err)
	}
	return <feature>FromRow(row), nil
}

func (r *postgresRepo) Delete(ctx context.Context, id string) error {
	rows, err := r.q.Delete<Feature>(ctx, id)
	if err != nil {
		return fmt.Errorf("<feature>: delete: %w", err)
	}
	if rows == 0 {
		return <feature>NotFound(id)
	}
	return nil
}

func <feature>FromRow(r sqlcgen.<Feature>) <Feature> {
	return <Feature>{
		ID:        r.ID,
		UserID:    r.UserID,
		Name:      r.Name,
		CreatedAt: r.CreatedAt,
		UpdatedAt: r.UpdatedAt,
	}
}
```

**Rules:**

- Repository interface uses domain types, not sqlc types
- `pgx.ErrNoRows` → domain sentinel via `<feature>NotFound(id)`
- Error wrapping: `fmt.Errorf("<feature>: <operation>: %w", err)`
- `identity.NewID()` for UUID generation
- `*FromRow()` function converts sqlc row → domain type
- Delete uses `:execrows` sqlc annotation, checks `rows == 0`

### Transaction Pattern

When a repo method needs atomicity across multiple writes:

```go
func (r *postgresRepo) CompleteThingTx(ctx context.Context, c Completion) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return fmt.Errorf("<feature>: begin tx: %w", err)
	}
	defer r.rollback(ctx, tx)

	q := r.q.WithTx(tx)

	// ... multiple q.Method(ctx, ...) calls ...

	return tx.Commit(ctx)
}

func (r *postgresRepo) rollback(ctx context.Context, tx pgx.Tx) {
	if err := tx.Rollback(ctx); err != nil && !errors.Is(err, pgx.ErrTxClosed) {
		r.lgr.Error(ctx, "<feature>: tx rollback failed", "err", err)
	}
}
```

When using transactions, the repo struct needs `pool *pgxpool.Pool` in addition to `q *sqlcgen.Queries`, and `lgr logger.Logger` for rollback logging.

---

## File: service.go

```go
package <feature>

import (
	"context"
	"errors"

	"github.com/jimnguyendev/myvocap/server/internal/db"
	"github.com/jimnguyendev/myvocap/server/internal/logger"
)

var Err<Feature>NotFound = errors.New("<feature> not found")

// Service defines the <feature> business operations.
type Service interface {
	List(ctx context.Context, userID string, page db.Page) ([]<Feature>, int, error)
	Get(ctx context.Context, userID, id string) (<Feature>, error)
	Create(ctx context.Context, userID string, in Create<Feature>Input) (<Feature>, error)
	Update(ctx context.Context, userID, id string, in Update<Feature>Input) (<Feature>, error)
	Delete(ctx context.Context, userID, id string) error
}

type service struct {
	repo Repository
	lgr  logger.Logger
}

func NewService(repo Repository, lgr logger.Logger) Service {
	return &service{repo: repo, lgr: lgr}
}

func (s *service) List(ctx context.Context, userID string, page db.Page) ([]<Feature>, int, error) {
	return s.repo.List(ctx, userID, page)
}

func (s *service) Get(ctx context.Context, userID, id string) (<Feature>, error) {
	item, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return <Feature>{}, err
	}
	// Ownership check: return not-found instead of forbidden (don't leak existence)
	if item.UserID != userID {
		return <Feature>{}, <feature>NotFound(id)
	}
	return item, nil
}

func (s *service) Create(ctx context.Context, userID string, in Create<Feature>Input) (<Feature>, error) {
	return s.repo.Create(ctx, userID, in)
}

func (s *service) Update(ctx context.Context, userID, id string, in Update<Feature>Input) (<Feature>, error) {
	existing, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return <Feature>{}, err
	}
	if existing.UserID != userID {
		return <Feature>{}, <feature>NotFound(id)
	}
	return s.repo.Update(ctx, id, in)
}

func (s *service) Delete(ctx context.Context, userID, id string) error {
	existing, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return err
	}
	if existing.UserID != userID {
		return <feature>NotFound(id)
	}
	return s.repo.Delete(ctx, id)
}
```

**Rules:**

- Service methods receive `userID` from handler (which gets it from middleware)
- Ownership check: return `NotFound` not `Forbidden` — don't leak entity existence
- Semantic validation (quotas, state checks) happens here
- Sentinel errors declared at package level or in `errors.go`

### With Cache + Events (full version)

```go
type service struct {
	repo     Repository
	cache    *cache.Cache
	eventBus event.Publisher
	lgr      logger.Logger
}

func NewService(repo Repository, c *cache.Cache, eb event.Publisher, lgr logger.Logger) Service {
	return &service{repo: repo, cache: c, eventBus: eb, lgr: lgr}
}

// Get with cache-aside
func (s *service) Get(ctx context.Context, id string) (<Feature>, error) {
	if s.cache != nil {
		return cache.FetchOne(ctx, s.cache, cacheKey(id), func(ctx context.Context) (<Feature>, error) {
			return s.repo.FindByID(ctx, id)
		}, cacheTTL)
	}
	return s.repo.FindByID(ctx, id)
}

// Create with event publishing
func (s *service) Create(ctx context.Context, in Create<Feature>Input) (<Feature>, error) {
	item, err := s.repo.Create(ctx, in)
	if err != nil {
		return <Feature>{}, err
	}
	if err := s.eventBus.PublishAsync(ctx, event.NewEvent(Topic<Feature>Created, item)); err != nil {
		s.lgr.Warn(ctx, "publish event failed", "topic", Topic<Feature>Created, "err", err)
	}
	return item, nil
}

// Update with cache invalidation + event
func (s *service) Update(ctx context.Context, id string, in Update<Feature>Input) (<Feature>, error) {
	item, err := s.repo.Update(ctx, id, in)
	if err != nil {
		return <Feature>{}, err
	}
	if s.cache != nil {
		if err := cache.Delete(ctx, s.cache, cacheKey(id)); err != nil {
			s.lgr.Warn(ctx, "cache delete failed", "key", cacheKey(id), "err", err)
		}
		cache.Forget(s.cache, cacheKey(id))
	}
	if err := s.eventBus.PublishAsync(ctx, event.NewEvent(Topic<Feature>Updated, item)); err != nil {
		s.lgr.Warn(ctx, "publish event failed", "topic", Topic<Feature>Updated, "err", err)
	}
	return item, nil
}
```

---

## File: handler.go

```go
package <feature>

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/jimnguyendev/myvocap/server/internal/httpresponse"
	"github.com/jimnguyendev/myvocap/server/internal/logger"
	"github.com/jimnguyendev/myvocap/server/internal/middleware"
	"github.com/jimnguyendev/myvocap/server/pkg/ginx"
)

type Handler struct {
	svc Service
	lgr logger.Logger
}

func NewHandler(svc Service, lgr logger.Logger) *Handler {
	return &Handler{svc: svc, lgr: lgr}
}

func (h *Handler) resolveUser(c *gin.Context) (string, bool) {
	uid, ok := middleware.LocalUserIDFromCtx(c.Request.Context())
	if !ok {
		httpresponse.WriteError(c, h.lgr,
			httpresponse.Unauthorized("MISSING_USER_CONTEXT", "missing user context", nil))
		return "", false
	}
	return uid, true
}

func (h *Handler) List(c *gin.Context) {
	uid, ok := h.resolveUser(c)
	if !ok {
		return
	}
	page := ginx.QueryPage(c, 100)
	items, total, err := h.svc.List(c.Request.Context(), uid, page)
	if err != nil {
		writeError(c, h.lgr, err)
		return
	}
	httpresponse.Paginated(c, items, httpresponse.Pagination{
		Total:   int64(total),
		Page:    page.Number,
		PerPage: page.Size,
	})
}

func (h *Handler) Get(c *gin.Context) {
	uid, ok := h.resolveUser(c)
	if !ok {
		return
	}
	item, err := h.svc.Get(c.Request.Context(), uid, c.Param("id"))
	if err != nil {
		writeError(c, h.lgr, err)
		return
	}
	httpresponse.Success(c, item)
}

func (h *Handler) Create(c *gin.Context) {
	uid, ok := h.resolveUser(c)
	if !ok {
		return
	}
	var in Create<Feature>Input
	if err := c.ShouldBindJSON(&in); err != nil {
		httpresponse.Error(c, http.StatusBadRequest, err)
		return
	}
	item, err := h.svc.Create(c.Request.Context(), uid, in)
	if err != nil {
		writeError(c, h.lgr, err)
		return
	}
	httpresponse.Success(c, item)
}

func (h *Handler) Update(c *gin.Context) {
	uid, ok := h.resolveUser(c)
	if !ok {
		return
	}
	var in Update<Feature>Input
	if err := c.ShouldBindJSON(&in); err != nil {
		httpresponse.Error(c, http.StatusBadRequest, err)
		return
	}
	item, err := h.svc.Update(c.Request.Context(), uid, c.Param("id"), in)
	if err != nil {
		writeError(c, h.lgr, err)
		return
	}
	httpresponse.Success(c, item)
}

func (h *Handler) Delete(c *gin.Context) {
	uid, ok := h.resolveUser(c)
	if !ok {
		return
	}
	if err := h.svc.Delete(c.Request.Context(), uid, c.Param("id")); err != nil {
		writeError(c, h.lgr, err)
		return
	}
	httpresponse.Success(c, nil)
}
```

**Rules:**

- Handler has exactly two fields: `svc Service` and `lgr logger.Logger`
- `resolveUser()` helper reads user UUID from middleware context
- Handlers that don't need user context (e.g., `example` feature) skip `resolveUser`
- Syntactic validation only: `c.ShouldBindJSON` → 400
- Domain errors: `writeError(c, h.lgr, err)` — one line, no if-else chain
- Always use `c.Request.Context()` when calling service, never `c` itself
- `c.Param("id")` for URL params, `ginx.QueryPage(c, max)` for pagination

---

## File: routes.go

```go
package <feature>

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func RegisterRoutes(rg *gin.RouterGroup, h *Handler) {
	v1 := rg.Group("/api/v1")
	v1.Handle(http.MethodGet, "/<feature>s", h.List)
	v1.Handle(http.MethodPost, "/<feature>s", h.Create)
	v1.Handle(http.MethodGet, "/<feature>s/:id", h.Get)
	v1.Handle(http.MethodPut, "/<feature>s/:id", h.Update)
	v1.Handle(http.MethodDelete, "/<feature>s/:id", h.Delete)
}
```

**Rules:**

- Use `http.Method*` constants, not string literals
- Use `v1.Handle()`, not `v1.GET()` / `v1.POST()`
- Plural resource names in URLs
- Nested resources: `/api/v1/<parent>s/:id/<child>s`

---

## File: provider.go

```go
package <feature>

import (
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/jimnguyendev/myvocap/server/internal/logger"
)

type Deps struct {
	Postgres *pgxpool.Pool
	Lgr      logger.Logger
}

func Provide(d Deps) *Handler {
	repo := NewPostgresRepository(d.Postgres)
	svc := NewService(repo, d.Lgr)
	return NewHandler(svc, d.Lgr)
}
```

### With Cache + Events

```go
type Deps struct {
	Postgres *pgxpool.Pool
	Cache    *cache.Cache
	EventBus *event.Bus
	Lgr      logger.Logger
}

func Provide(d Deps) *Handler {
	eb := event.Noop()
	if d.EventBus != nil {
		eb = d.EventBus
	}

	repo := NewPostgresRepository(d.Postgres)
	svc := NewService(repo, d.Cache, eb, d.Lgr)
	registerListeners(d.EventBus, d.Lgr)
	return NewHandler(svc, d.Lgr)
}
```

**Rules:**

- `Deps` lists only what this feature needs — no kitchen sink
- `Provide()` is the single wiring point: repo → service → handler
- Cache/EventBus may be nil — degrade gracefully
- `registerListeners()` called here if events are used
- Returns `*Handler` (not interface) — only service/repo use interfaces

---

## File: errors.go

```go
package <feature>

import (
	"context"
	"errors"
	"fmt"

	"github.com/gin-gonic/gin"

	"github.com/jimnguyendev/myvocap/server/internal/httpresponse"
)

func <feature>NotFound(id string) error {
	return fmt.Errorf("%w: %s", Err<Feature>NotFound, id)
}

type errorLogger interface {
	Error(ctx context.Context, msg string, kvs ...any)
}

func writeError(c *gin.Context, lgr errorLogger, err error) {
	httpresponse.WriteError(c, lgr, toAppError(err))
}

func toAppError(err error) error {
	switch {
	case err == nil:
		return nil
	case errors.Is(err, Err<Feature>NotFound):
		return httpresponse.NotFound("<FEATURE>_NOT_FOUND", "<feature> not found", err)
	}
	return err
}
```

**Rules:**

- `errorLogger` is a narrow interface (not the full logger) — keeps errors.go decoupled
- `writeError` is the ONLY way handlers send error responses (except syntactic 400s)
- `toAppError` maps sentinels to `httpresponse.*` constructors
- Unmapped errors return as-is → `WriteError` treats them as 500
- AppError code format: `SCREAMING_SNAKE_CASE` (e.g., `DECK_NOT_FOUND`, `QUOTA_EXCEEDED`)
- Add new `case` lines as new sentinel errors are introduced

### Adding more sentinels

```go
var (
	Err<Feature>NotFound     = errors.New("<feature> not found")
	ErrQuotaExceeded         = errors.New("<feature> quota exceeded")
	ErrAlreadyExists         = errors.New("<feature> already exists")
)

func toAppError(err error) error {
	switch {
	case err == nil:
		return nil
	case errors.Is(err, Err<Feature>NotFound):
		return httpresponse.NotFound("<FEATURE>_NOT_FOUND", "<feature> not found", err)
	case errors.Is(err, ErrQuotaExceeded):
		return httpresponse.Unprocessable("<FEATURE>_QUOTA_EXCEEDED", "quota exceeded", err)
	case errors.Is(err, ErrAlreadyExists):
		return httpresponse.Conflict("<FEATURE>_ALREADY_EXISTS", "<feature> already exists", err)
	}
	return err
}
```

### AppError status mapping

| Sentinel meaning | `httpresponse.*` constructor | HTTP status |
|---|---|---|
| Not found | `NotFound(code, msg, cause)` | 404 |
| Bad input (semantic) | `BadRequest(code, msg, cause)` | 400 |
| Unauthorized | `Unauthorized(code, msg, cause)` | 401 |
| Forbidden | `Forbidden(code, msg, cause)` | 403 |
| Already exists / conflict | `Conflict(code, msg, cause)` | 409 |
| Business rule violation / quota | `Unprocessable(code, msg, cause)` | 422 |
| Upstream failure | `BadGateway(code, msg, cause)` | 502 |

---

## File: events.go (optional)

```go
package <feature>

const (
	Topic<Feature>Created = "<feature>:created"
	Topic<Feature>Updated = "<feature>:updated"
	Topic<Feature>Deleted = "<feature>:deleted"
)
```

**Rules:**

- Topic format: `<feature>:<action>` (colon-separated, lowercase)
- Only create topics that the feature actually publishes

---

## File: listeners.go (optional)

```go
package <feature>

import (
	"context"

	"github.com/jimnguyendev/myvocap/server/internal/event"
	"github.com/jimnguyendev/myvocap/server/internal/logger"
)

func registerListeners(bus *event.Bus, lgr logger.Logger) {
	if bus == nil {
		return
	}

	bus.Subscribe(Topic<Feature>Created, "<feature>.on-created", func(ctx context.Context, e event.Event) error {
		created := e.Payload.(<Feature>)
		lgr.Info(ctx, "<feature> created", "id", created.ID)
		return nil
	})
}
```

**Rules:**

- Guard `bus == nil` at the top
- Subscriber ID format: `<feature>.<handler-name>` (dot-separated)
- Type-assert payload: `e.Payload.(<Type>)` — bus guarantees the type
- Called from `Provide()`, runs for process lifetime

---

## SQL Queries: internal/db/queries/<feature>.sql

```sql
-- name: Count<Feature>s :one
SELECT count(*) FROM <feature>s WHERE user_id = $1;

-- name: List<Feature>s :many
SELECT id, user_id, name, created_at, updated_at
FROM <feature>s
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;

-- name: Get<Feature> :one
SELECT id, user_id, name, created_at, updated_at
FROM <feature>s
WHERE id = $1;

-- name: Create<Feature> :one
INSERT INTO <feature>s (id, user_id, name)
VALUES ($1, $2, $3)
RETURNING id, user_id, name, created_at, updated_at;

-- name: Update<Feature> :one
UPDATE <feature>s
SET name = $2, updated_at = now()
WHERE id = $1
RETURNING id, user_id, name, created_at, updated_at;

-- name: Delete<Feature> :execrows
DELETE FROM <feature>s WHERE id = $1;
```

**Rules:**

- `:one` for single-row queries, `:many` for lists, `:execrows` for delete (to check 0 rows)
- Always `RETURNING` full row for `:one` inserts/updates
- `ORDER BY created_at DESC` as default sort
- Run `sqlc generate` after writing queries
