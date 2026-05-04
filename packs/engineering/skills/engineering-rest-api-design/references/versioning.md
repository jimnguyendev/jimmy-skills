# API Versioning Strategies

## When to Version

Version when you make **breaking changes**:

- Renaming a field
- Changing a field's type or format (string → integer, date format change)
- Changing field validation rules (max length, regex)
- Making an optional field required
- Removing a field or endpoint

Non-breaking changes (adding optional fields, new endpoints) generally do not require a new version.

## URL Path Versioning

Include the version directly in the URL:

```
GET /v1/orders
GET /v2/orders
```

**Pros:** Visible, simple, easy to route and cache.
**Cons:** Every breaking change requires a new URL path. Old versions must be maintained.

### Channel-Based Versioning

Extend URL versioning with release channels for staged rollout:

```
/v1/stable/orders    # or simply /v1/orders
/v1/beta/orders      # or /v1beta/orders
/v1/alpha/orders     # or /v1alpha/orders
```

**Stages:**

- **v1alpha** — early release, internal testing only. Expect instability and incomplete features.
- **v1beta** — broader testing, closer to final. May still change based on feedback.
- **v1 (stable)** — fully tested, ready for production use.

## Header Versioning

Version information is sent in HTTP request headers:

```
GET /orders
Headers:
  Api-Version: 2
  Accept: application/vnd.myapi.v2+json
```

**Pros:** URLs stay clean, flexible switching.
**Cons:** Version is hidden — harder to discover, debug, and share. Clients must remember to set headers.

## Query Parameter Versioning

Version included as a query parameter:

```
GET /orders?version=2.0
```

**Pros:** Simple to implement, flexible.
**Cons:** Easy to forget, mixes API metadata with business query parameters.

## Decision Guide

| Scenario | Recommendation |
|----------|---------------|
| Public API with many external consumers | URL path versioning |
| API with staged release process | Channel-based versioning |
| Internal API, all clients controlled | May not need versioning |
| API gateway handles routing | Header versioning works well |
| Rapid prototyping | Query parameter for flexibility |

## Best Practices

1. **Support at most 2 active versions** — maintaining more is costly.
2. **Communicate deprecation timelines** — give consumers time to migrate.
3. **Use `Sunset` header** to signal deprecation: `Sunset: Sat, 01 Jun 2025 00:00:00 GMT`.
4. **Log version usage** — know which consumers are on which version before sunsetting.
5. **Default to latest stable** — if no version specified, serve the latest stable version.
