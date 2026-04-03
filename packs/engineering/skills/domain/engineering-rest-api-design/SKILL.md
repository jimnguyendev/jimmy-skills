---
name: engineering-rest-api-design
description: "REST API design conventions covering URL structure, HTTP methods, pagination, async patterns, idempotency, error envelopes, and API documentation standards. Use when designing new endpoints, reviewing API contracts, or establishing API guidelines before implementation in any language."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a senior API architect. Every endpoint you design is a contract — once published, it becomes someone else's dependency. Design for the consumer first, optimize for the maintainer second.

**Modes:**

- **Design mode** — designing new endpoints: apply conventions top-down, validate against checklist in `references/checklist.md`.
- **Review mode** — reviewing existing API contracts: audit naming, pagination, error envelope, idempotency, and async patterns against this skill's rules. Flag violations with severity (breaking / inconsistent / style).
- **Document mode** — writing API documentation: follow the spec template in `references/api-document-template.md`.

# REST API Design

## Mindset

1. **Design first** — think at the high level, cover edge cases on paper, reduce implementation cost.
2. **Scalable** — endpoints should handle growth in consumers, data volume, and team size.
3. **Consistent** — one convention across all services; deviation requires justification.
4. **Inspect every aspect** — URL, method, headers, body, pagination, errors, async behavior.
5. **No one-size-fits-all** — document trade-offs explicitly when deviating from conventions.

## HTTP Methods

| Method | Operation | Safe | Idempotent |
|--------|-----------|------|------------|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Replace entirely | No | Yes |
| PATCH | Update partially | No | Potentially |
| DELETE | Remove / disable | No | Yes |

**Safety** means the method does not alter server state. **Idempotency** means sending the same request multiple times produces the same result.

PATCH is *potentially* idempotent: if it consistently sets a field to the same value, it is idempotent. If it increments a counter or appends to a list, it is not. Design PATCH operations to be idempotent whenever possible.

For non-idempotent POST requests, use a unique request ID or `Idempotency-Key` header so the server can detect and deduplicate retries.

## URL Conventions

### Rules

1. **Nouns, not verbs** — the resource is the noun, the method is the verb.
2. **Plural nouns** — `/users`, not `/user`.
3. **Nesting for relationships** — `/articles/{article_id}/comments`.
4. **Versioning in path** — `/api/v1/...`.
5. **Slug-case for URLs** — `/order-service/v1/orders`.
6. **snake_case for request and response body** — `{ "debit_account": "acc01" }`.

### Singular vs Plural

Use plural by default. Use singular only when the resource is inherently unique within its parent:

```
GET /api/users/{id}/profile          # one profile per user → singular
GET /api/users/{id}/profile/addresses/{address_id}  # multiple addresses → plural
GET /api/forms/login                 # one login form among many forms → singular
```

### Custom Actions

When CRUD methods are insufficient (restore, publish, archive), use one of:

**Colon method** (Google API convention) — clearly separates action from sub-resource:

```
POST /files/{id}:restore
POST /v1/{resource}:setIamPolicy
```

**Slash method** — simpler but risks confusion with sub-resources:

```
POST /files/{id}/restore
```

Prefer the colon method when clarity matters. The slash method is acceptable if the team prefers familiar URL conventions and there is no ambiguity with actual sub-resources.

### Examples

```
POST   /order-service/v1/orders
GET    /order-service/v1/orders/145
PATCH  /user-service/v1/users/34
DELETE /user-service/v1/users/34
```

## Pagination

Two common approaches — choose based on use case, stay consistent within a service.

### Page + Size

```
GET /users?page=0&size=10
```

- Best for: management portals, admin dashboards.
- Must document: whether page starts at 0 or 1.

### Offset + Limit

```
GET /users?offset=0&limit=10
```

- Best for: infinite scroll, newsfeeds, log streams.

### Known Problems

1. **Performance on large datasets** — `OFFSET N` scans and discards N rows.
2. **Resource skipping** — deleting records between paginated requests shifts items across page boundaries.

### Solutions

**Cursor-based pagination** — use the last seen ID as a cursor:

```sql
SELECT * FROM users WHERE id > :last_id ORDER BY id LIMIT 10;
```

**Deferred join** — fetch IDs first, then join:

```sql
SELECT * FROM (
  SELECT id FROM users ORDER BY id LIMIT 100, 10
) a JOIN users b ON a.id = b.id;
```

See `references/pagination-patterns.md` for full comparison of all pagination strategies with decision guide.

## Filtering

Use query parameters to narrow results. Multiple filters combine with AND logic:

```
GET /products?price=20&brand=Nike
GET /orders?status=pending&created_after=2024-01-01
```

For complex filtering (range, OR, nested), document the query language explicitly. Never pass filter values directly into SQL — always parameterize.

## Sorting

Three common conventions — pick one per API, stay consistent:

```
# Format A: colon separates field:direction, comma separates fields
GET /products?sort=price:asc,name:desc

# Format B: prefix +/- for direction
GET /products?sort=+price,-name

# Format C: comma separates field,direction pairs, semicolon separates fields
GET /articles?sort=publish_date,asc;title,desc
```

Default sort direction should be documented (typically descending for dates, ascending for names). **Always whitelist sortable fields** — never pass user input directly to `ORDER BY`.

## Relationship Endpoints

### One-to-Many

```
GET /articles/{article_id}/comments
```

### Many-to-Many

```
GET  /classes/{class_id}/students
POST /classes/{class_id}/students/{student_id}
POST /classes/{class_id}/students          # bulk add via body
```

Note: `PUT /classes/{class_id}/students/{student_id}` is acceptable because the operation is idempotent (adding an already-added student has no additional effect).

## Async API Pattern

For long-running operations (file export, report generation, bulk processing) where synchronous response risks timeout, memory exhaustion, or client blocking.

### Job-based Flow

```
# 1. Initiate the job
POST /products/jobs/export?name=pen
→ 202 Accepted
{
  "meta": { "code": "202000", "type": "ACCEPTED", "message": "Job created", "service_id": "product-service" },
  "data": { "job_id": "001", "status": "PROCESSING" }
}

# 2. Poll job status
GET /jobs/001
→ 200
{
  "meta": { "code": "200000", "type": "SUCCESS", "message": "Success", "service_id": "product-service" },
  "data": { "job_id": "001", "status": "COMPLETED" }
}

# 3. Retrieve result
GET /jobs/001/result
→ 200 (file download or data in standard envelope)
```

### Polling vs Webhook

| Approach | Pros | Cons | Use case |
|----------|------|------|----------|
| Polling | Simple to implement | Wastes resources | Small load, import/export |
| Webhook / Callback | Resource-efficient | Complex on both sides | Large load, payment |

## Versioning

See `references/versioning.md` for full comparison. Summary:

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | `/v1/orders` | Visible, simple | New URL per version |
| Channel | `/v1/beta/orders` | Staged rollout | More paths to manage |
| Header | `Api-Version: 2` | URL stays clean | Hidden, easy to miss |
| Query param | `/orders?version=2` | Flexible | Easy to forget |

**Default recommendation:** URL path versioning (`/v1/`). Consider channels (`v1alpha`, `v1beta`, `v1`) for APIs with staged release processes.

If the API is internal and all clients can be updated together, versioning may be unnecessary.

## Rate Limiting

Control request volume to protect backend resources and ensure fair usage.

**Response for exceeded limits:** return `429 Too Many Requests`.

**Inform clients via headers:**

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 500
X-RateLimit-Reset: 1588377600
Retry-After: 120
```

## Idempotency

**Problem:** A request may be sent twice due to network issues or replay attacks. Critical for payment, order, and financial operations.

**Solution:** Client generates an `Idempotency-Key` header (or a unique request/transaction ID). Server enforces uniqueness via a unique constraint in the database. On duplicate, server returns the original response — not an error.

```
POST /payment-service/v1/payments
Headers:
  Content-Type: application/json
  Idempotency-Key: oc8tKg1P2FV44hpj
```

## Response Envelope

Standard envelope structure for all API responses:

```json
{
  "meta": {
    "code": "200000",
    "type": "SUCCESS",
    "message": "Success",
    "service_id": "payment-service",
    "extra_meta": {}
  },
  "data": { ... }
}
```

Error responses use the same envelope with `"data": null`:

```json
{
  "meta": {
    "code": "400001",
    "type": "INSUFFICIENT_DEBIT_AMOUNT",
    "message": "Debit account has an insufficient amount of balance",
    "service_id": "payment-service",
    "extra_meta": {}
  },
  "data": null
}
```

## API Documentation

Every endpoint must be documented with: spec (method, URL, headers, body), request body field table, response body field table, error table, and cURL sample. See `references/api-document-template.md` for the full template.

## Cross-References

- For backend implementation of these patterns, start with `jimmy-skills@backend-core`.
- For Go-specific HTTP handler details, use `jimmy-skills@backend-go-code-style`.
- For frontend API integration, use `jimmy-skills@frontend-core`.
- For error handling conventions in Go, use `jimmy-skills@backend-go-error-handling`.
- For database query patterns (pagination SQL), use `jimmy-skills@backend-go-database`.

## External Sources

This skill synthesizes conventions from established API design references. Official documentation remains authoritative:

- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Stripe API Reference](https://stripe.com/docs/api) (idempotency patterns)
- [REST API Best Practices — freeCodeCamp](https://www.freecodecamp.org/news/rest-api-best-practices-rest-endpoint-design-examples/)
