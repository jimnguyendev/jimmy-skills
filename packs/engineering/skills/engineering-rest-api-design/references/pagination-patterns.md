# Pagination Patterns

## Comparison

| Pattern | SQL | Best for | Drawbacks |
|---------|-----|----------|-----------|
| Page + Size | `LIMIT size OFFSET (page * size)` | Admin portals, small datasets | Performance degrades at high page numbers |
| Offset + Limit | `LIMIT limit OFFSET offset` | Infinite scroll, feeds | Same offset performance issue + resource skipping |
| Cursor-based | `WHERE id > :cursor LIMIT :limit` | Large datasets, real-time feeds | Cannot jump to arbitrary page |
| Keyset | `WHERE (col1, col2) > (:v1, :v2) ORDER BY col1, col2 LIMIT :limit` | Multi-column sort | Complex for composite keys |

## Problem: Offset Performance

```sql
-- Slow: DB scans and discards 100,000 rows
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 100000;
```

The database must read and discard all offset rows before returning the requested page.

## Problem: Resource Skipping

1. Client fetches page 1 (items 1-10).
2. Another client deletes items 3 and 7 from page 1.
3. Client fetches page 2 — items that were 11-12 shifted to page 1, so the client never sees them.

## Solution: Cursor-Based Pagination

```sql
-- Fast: index seek directly to the cursor position
SELECT * FROM users
WHERE id > :last_seen_id
ORDER BY id
LIMIT 10;
```

Response includes the cursor for the next page, wrapped in the standard envelope:

```json
{
  "meta": {
    "code": "200000",
    "type": "SUCCESS",
    "message": "Success",
    "service_id": "user-service",
    "extra_meta": {
      "next_cursor": "user_abc123",
      "has_more": true
    }
  },
  "data": [...]
}
```

Pagination metadata (`next_cursor`, `has_more`) goes in `extra_meta` to keep the envelope consistent across all endpoints.

## Solution: Deferred Join

When cursor-based is not possible (e.g., complex sorting), use deferred join to minimize row fetching:

```sql
SELECT b.* FROM (
  SELECT id FROM users ORDER BY created_at DESC LIMIT 100, 10
) a JOIN users b ON a.id = b.id;
```

The subquery scans only the indexed `id` column for offset, then joins to fetch full rows only for the 10 results.

## Choosing a Pattern

- **Small dataset, admin UI** → page/size (simplicity wins).
- **Feed, timeline, log stream** → cursor-based (performance and consistency).
- **Search results with sorting** → deferred join or keyset (performance with flexibility).
- **Public API** → cursor-based (protects backend from deep pagination abuse).

## Response Headers

Optionally include pagination metadata in headers as well (useful for clients that need it before parsing body):

```
X-Total-Count: 1523
X-Page-Size: 10
Link: </users?cursor=abc123&limit=10>; rel="next"
```

Note: always include pagination info in the response body (`extra_meta`) as the primary source. Headers are supplementary.
