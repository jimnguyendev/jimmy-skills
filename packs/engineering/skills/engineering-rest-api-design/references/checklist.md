# REST API Design Checklist

Use this checklist when designing or reviewing API endpoints.

## URL Structure

- [ ] Resource names are plural nouns (`/users`, not `/user`)
- [ ] Singular nouns used only for inherently unique sub-resources (`/users/{id}/profile`)
- [ ] No verbs in URL path (use HTTP methods instead)
- [ ] Custom actions use colon (`:restore`) or slash method — not CRUD verbs in URL
- [ ] Relationships expressed via nesting (`/articles/{id}/comments`)
- [ ] URL uses slug-case (`/order-service`, not `/orderService`)
- [ ] Version included in path (`/api/v1/...`)
- [ ] IDs in path are positional, not query params (`/users/34`, not `/users?id=34`)

## Request / Response Body

- [ ] Field names use snake_case (`debit_account`, not `debitAccount`)
- [ ] Request body documented with type, required flag, default, and description
- [ ] Response follows standard envelope (`meta` + `data`)
- [ ] Null `data` on error responses, never omitted

## HTTP Methods

- [ ] GET for reads (safe, idempotent)
- [ ] POST for creates — returns `200`, not `201` (not idempotent unless Idempotency-Key used)
- [ ] POST for batch reads by IDs (`POST /resources/batch` with `{ "ids": [...] }`)
- [ ] PUT for all updates — full or partial, no PATCH (idempotent)
- [ ] DELETE for removal or disabling (idempotent)
- [ ] Method choice matches safety and idempotency properties

## Pagination

- [ ] Pagination strategy chosen and documented (page/size OR offset/limit OR cursor)
- [ ] Page start index documented (0 or 1)
- [ ] Default page size defined and enforced
- [ ] Maximum page size capped
- [ ] Cursor-based pagination used for large or mutable datasets

## Filtering

- [ ] Filter parameters documented with types and valid values
- [ ] Multiple filters combine with AND logic by default
- [ ] Filter values parameterized — never interpolated into SQL
- [ ] Complex filtering query language documented (if applicable)

## Sorting

- [ ] Sort format documented (`field:asc` or `+field`)
- [ ] Sortable fields whitelisted — no arbitrary column sorting
- [ ] Default sort order documented

## Async Operations

- [ ] Long-running operations return 202 with job reference
- [ ] Job status endpoint exists (`GET /jobs/{id}`)
- [ ] Job result endpoint exists (`GET /jobs/{id}/result`)
- [ ] Delivery method chosen: polling or webhook
- [ ] Webhook retry and failure handling documented (if webhook)

## Idempotency

- [ ] Idempotency-Key header required for create operations on critical resources (payment, order)
- [ ] Server enforces uniqueness via DB constraint
- [ ] Duplicate request returns original response, not error

## Versioning

- [ ] Versioning strategy chosen (URL path, header, query param, or none)
- [ ] Breaking changes trigger a version bump
- [ ] Deprecation timeline communicated to consumers
- [ ] At most 2 active versions maintained

## Rate Limiting

- [ ] Rate limits defined and documented
- [ ] 429 status returned when limits exceeded
- [ ] `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers included
- [ ] `Retry-After` header included in 429 responses

## Error Handling

- [ ] All error codes and types documented in error table
- [ ] HTTP status codes used correctly (400 for client error, 401/403 for auth, 404 for not found, 500 for server error)
- [ ] Error `type` is machine-readable constant (`INSUFFICIENT_DEBIT_AMOUNT`)
- [ ] Error `message` is human-readable explanation
- [ ] Service ID included in error envelope for tracing

## Documentation

- [ ] API spec includes: method, URL, headers, request body, response body
- [ ] Field-level documentation with type, required, default, description
- [ ] Error table with status code, internal code, type, description
- [ ] cURL sample provided
- [ ] Authentication requirements documented
