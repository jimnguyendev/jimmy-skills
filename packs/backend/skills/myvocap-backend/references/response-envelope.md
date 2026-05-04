# Response Envelope

All API responses use the standard envelope from `internal/httpresponse/`.

## Envelope Shape

```json
{
  "meta": {
    "code": "200000",
    "type": "SUCCESS",
    "message": "Success",
    "service_id": "myvocap",
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "pagination": {
      "total": 42,
      "page": 1,
      "per_page": 20
    }
  },
  "data": { ... }
}
```

### Meta Fields

| Field | Format | Description |
|---|---|---|
| `code` | `"{status}000"` | HTTP status + "000" suffix (e.g., `"200000"`, `"404000"`) |
| `type` | `SCREAMING_SNAKE` | Status category or error code |
| `message` | human string | User-safe message |
| `service_id` | string | Set once at startup via `httpresponse.SetServiceID("myvocap")` |
| `request_id` | UUID | From `X-Request-ID` header (injected by `middleware.RequestID`) |
| `pagination` | object or omitted | Only present on paginated list responses |

### Type Values for Success

| HTTP Status | Type |
|---|---|
| 200 | `SUCCESS` |

### Type Values for Errors

For errors mapped through `toAppError()`, the type is the `Code` field from `AppError`:

| Sentinel | AppError Code (= `meta.type`) | HTTP Status |
|---|---|---|
| `ErrDeckNotFound` | `DECK_NOT_FOUND` | 404 |
| `ErrCardNotFound` | `CARD_NOT_FOUND` | 404 |
| `ErrQuotaExceeded` | `DECK_QUOTA_EXCEEDED` | 422 |
| unmapped error | `INTERNAL_ERROR` | 500 |

For syntactic errors (handler `ShouldBindJSON` failures), the type is the generic status type:

| HTTP Status | Type |
|---|---|
| 400 | `BAD_REQUEST` |
| 401 | `UNAUTHORIZED` |
| 429 | `TOO_MANY_REQUESTS` |
| 500 | `INTERNAL_SERVER_ERROR` |

## Response Helpers

### Success (200)

```go
httpresponse.Success(c, data)
```

```json
{
  "meta": { "code": "200000", "type": "SUCCESS", "message": "Success", ... },
  "data": { "id": "abc", "name": "hello" }
}
```

### Paginated List (200 + pagination)

```go
httpresponse.Paginated(c, items, httpresponse.Pagination{
    Total:   int64(total),
    Page:    page.Number,
    PerPage: page.Size,
})
```

```json
{
  "meta": {
    "code": "200000",
    "type": "SUCCESS",
    "message": "Success",
    "service_id": "myvocap",
    "request_id": "...",
    "pagination": { "total": 42, "page": 1, "per_page": 20 }
  },
  "data": [
    { "id": "abc", "name": "hello" },
    { "id": "def", "name": "world" }
  ]
}
```

### Syntactic Error (400)

```go
// In handler, after ShouldBindJSON fails
httpresponse.Error(c, http.StatusBadRequest, err)
```

```json
{
  "meta": { "code": "400000", "type": "BAD_REQUEST", "message": "Key: 'Name' ...", ... },
  "data": null
}
```

Note: for 5xx, `Error()` replaces the message with generic `http.StatusText(code)` to avoid leaking internals.

### Domain Error (via writeError)

```go
// In handler
writeError(c, h.lgr, err)
```

This calls `toAppError(err)` → `httpresponse.WriteError(c, lgr, appError)`:

```json
{
  "meta": { "code": "404000", "type": "DECK_NOT_FOUND", "message": "deck not found", ... },
  "data": null
}
```

### Delete Success

```go
httpresponse.Success(c, nil)
```

```json
{
  "meta": { "code": "200000", "type": "SUCCESS", "message": "Success", ... },
  "data": null
}
```

## Handler Decision Tree

```
Is it a ShouldBindJSON error?
  ├─ YES → httpresponse.Error(c, http.StatusBadRequest, err)
  └─ NO  → Is it a service/repo error?
           ├─ YES → writeError(c, h.lgr, err)
           └─ NO  → Is it success?
                    ├─ List?  → httpresponse.Paginated(c, items, pagination)
                    └─ Other  → httpresponse.Success(c, data)
```

**Note:** All successful responses use `httpresponse.Success(c, data)` — including POST create. Per `jimmy-skills@engineering-rest-api-design`, POST returns 200 not 201.

## Setup

In `cmd/app/http.go`, before any routes:

```go
httpresponse.SetServiceID(cfg.General.ServiceName) // "myvocap"
```
