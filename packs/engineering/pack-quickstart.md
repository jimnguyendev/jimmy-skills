## Engineering Pack

3 skills for stack-agnostic engineering process — design thinking, API design, and performance optimization.

### Routing

- Use `engineering-design-thinking` **before** starting any large feature (multiple files, new concepts, migrations).
- Use `engineering-rest-api-design` when designing or reviewing REST API contracts.
- Use `engineering-perf-optimization-process` when performance is the constraint — requires measurable targets before any optimization.

### Skills

| Skill | When to use |
|---|---|
| `engineering-design-thinking` | Design-first process for large features. 5-gate framework: problem framing → context analysis → solution evaluation → architecture decision → skill routing. Use before implementation. |
| `engineering-rest-api-design` | REST API conventions: URL structure, HTTP methods (POST returns 200), pagination (cursor for large datasets), async patterns, idempotency, error envelope, rate limiting, versioning. Not user-invocable — triggers in design/review/document modes. |
| `engineering-perf-optimization-process` | Constraint-driven optimization. Prevents over-engineering by requiring: concrete targets → hot path identification → profiling evidence → simplest sufficient solution → rollback plan. Blocks "make it faster" without metrics. |

### Relationship to Other Packs

- Backend implementation details → `packs/backend/` (`backend-go-*` skills)
- These skills provide the **process** layer; implementation skills provide the **how**.
