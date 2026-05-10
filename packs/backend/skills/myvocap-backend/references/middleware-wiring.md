# Middleware Chain & Route Groups

## Middleware Stack (ordered)

```go
// cmd/app/http.go

engine.Use(gin.Recovery())                    // panic → 500 (built-in)
engine.Use(middleware.RequestID())            // generate/validate X-Request-ID
engine.Use(i18nManager.Middleware())          // locale detection
engine.Use(observe.GinTracer(...))            // OTel distributed tracing
engine.Use(middleware.Logging(lgr))           // structured request logging
engine.Use(middleware.CORS(...))              // CORS headers
engine.Use(middleware.BodyLimit(...))         // request size limit
```

Order matters. RequestID must run before Logging (injects `chain_id` for log correlation).

## Route Groups

```go
// --- protected: has auth, no local user UUID ---
protected := engine.Group("/",
    middleware.IdentitySignature(...),        // validate signed Prep headers
    middleware.RequireSignedIdentity(...),    // enforce signature in non-local env
    middleware.RateLimitIP(...),              // anonymous rate limit
    middleware.Auth(authenticator, lgr),      // Bearer token → prepID in ctx
)

// --- localUser: has auth + resolved local UUID ---
localUser := protected.Group("/",
    middleware.RequireLocalUser(...),         // prepID → local UUID in ctx
    middleware.RateLimitUser(...),            // user + mutation-specific limits
)
```

## Which Group to Use

| Feature needs... | Mount on | Example |
|---|---|---|
| No user identity | `engine` (public) | health check, docs |
| Auth (prepID) but no local UUID | `protected` | `user`, `example` |
| Local user UUID | `localUser` | `vocabulary`, `review`, `battle`, `gamification` |

**Rule:** If a handler calls `middleware.LocalUserIDFromCtx()`, it MUST be on `localUser`. If it doesn't need user UUID, use `protected`. Don't move routes to `localUser` "just in case".

## Wiring a New Feature

```go
// cmd/app/http.go

// 1. Create handler
thingHandler := thing.Provide(thing.Deps{
    Postgres: dbConn,
    Lgr:      lgr,
})

// 2. Register routes on the correct group
thing.RegisterRoutes(localUser, thingHandler)
```

## Consumer-Side Interface Pattern

When middleware needs data from a feature, it declares a narrow interface — the feature's service satisfies it without either importing the other.

### Example: LocalUserResolver

```go
// internal/middleware/local_user.go — declares the interface
type LocalUserResolver interface {
    LocalUserID(ctx context.Context, prepID int64, token string) (string, error)
}

func RequireLocalUser(r LocalUserResolver, lgr logger.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        prepID, ok := auth.UserIDFromCtx(c.Request.Context())
        if !ok {
            c.Abort()
            return
        }
        ctxTimeout, cancel := context.WithTimeout(c.Request.Context(), 10*time.Second)
        defer cancel()
        uid, err := r.LocalUserID(ctxTimeout, prepID, ExtractBearerToken(c))
        if err != nil {
            // handle error
            return
        }
        ctx := context.WithValue(c.Request.Context(), localUserCtxKey{}, uid)
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}
```

```go
// internal/feature/user/service.go — satisfies the interface (implicitly)
func (s *service) LocalUserID(ctx context.Context, prepID int64, token string) (string, error) {
    // ... lookup or create local user ...
}
```

```go
// cmd/app/http.go — wiring
localUser := protected.Group("/",
    middleware.RequireLocalUser(userBundle.Svc, lgr), // service satisfies interface
)
```

**Rules:**

- Interface declared in the CONSUMER package (middleware), not the provider (feature)
- Interface is narrow — only the methods the consumer needs
- Wiring happens in `cmd/app/http.go` — the only place that imports both
- No feature imports middleware; no middleware imports features

## Reading User Context in Handlers

```go
// Get local user UUID (set by RequireLocalUser middleware)
uid, ok := middleware.LocalUserIDFromCtx(c.Request.Context())

// Get Prep user ID (set by Auth middleware)
prepID, ok := auth.UserIDFromCtx(c.Request.Context())
```

### resolveUser Helper Pattern

Every handler that needs user UUID should have this helper:

```go
func (h *Handler) resolveUser(c *gin.Context) (string, bool) {
    uid, ok := middleware.LocalUserIDFromCtx(c.Request.Context())
    if !ok {
        httpresponse.WriteError(c, h.lgr,
            httpresponse.Unauthorized("MISSING_USER_CONTEXT", "missing user context", nil))
        return "", false
    }
    return uid, true
}
```

Then in each handler method:

```go
func (h *Handler) Get(c *gin.Context) {
    uid, ok := h.resolveUser(c)
    if !ok {
        return
    }
    // ... use uid ...
}
```

## Auth Middleware

```go
// middleware.Auth extracts Bearer token → validates via Authenticator → sets prepID in ctx
func Auth(a auth.Authenticator, lgr logger.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := ExtractBearerToken(c)
        ctxTimeout, cancel := context.WithTimeout(c.Request.Context(), 30*time.Second)
        defer cancel()
        ok, err := a.Valid(ctxTimeout, token, &userID)
        // ... set userID in context ...
    }
}
```

**Rule:** 30s timeout on auth validation. The authenticator has a circuit breaker (`auth.WithCircuitBreaker`) to fail fast when the auth service is down.

## Rate Limiting

Two layers:

```go
// IP-based (anonymous) — on protected group
middleware.RateLimitIP(rdb, cfg.RateLimit)

// User-based (authenticated) — on localUser group
middleware.RateLimitUser(rdb, cfg.RateLimit)
```

Rate limiter uses Redis fixed-window. Config in env vars (`RATE_LIMIT_*`).

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Handler calls `userSvc.LocalUserID(...)` directly | Use `middleware.LocalUserIDFromCtx(ctx)` — middleware already resolved it |
| Feature imports `internal/middleware` | Use consumer-side interface; wire in `cmd/app/http.go` |
| Middleware imports `internal/feature/user` | Declare narrow interface in middleware package |
| Route on `localUser` that never reads user UUID | Move to `protected` |
| Missing timeout on `LocalUserResolver` call | Always use `context.WithTimeout` |
