## Backend Pack

31 skills for Go backend development — from architecture to production debugging.

### Routing

- Start with `backend-core` when the problem is architectural, operational, or spans multiple concerns.
- Use `myvocap-backend` for MyVocab project-specific patterns (feature module template, error flow, DI wiring).
- Route into `backend-go-*` for Go implementation details after the boundary is clear.

### Architecture & Design

| Skill | When to use |
|---|---|
| `backend-core` | Architectural decisions, service boundaries, trade-offs, cross-service contracts |
| `backend-go-design-patterns` | Constructors, DI (Deps struct), functional options, resource lifecycle, resilience |
| `backend-go-project-layout` | Directory structure, feature-first vs layer-first, Go workspaces |
| `backend-go-structs-interfaces` | Interface design, composition, embedding, type assertions, dependency injection |
| `backend-go-naming` | Package, function, variable, error, and interface naming conventions |
| `backend-go-code-style` | Readability, comments, file organization, maintainability tradeoffs |

### Data & Persistence

| Skill | When to use |
|---|---|
| `backend-go-database` | sqlc/pgx/sqlx, parameterized queries, transactions, connection pool, migrations |
| `backend-go-data-structures` | Slices, maps, generics, container/, unsafe.Pointer, copy semantics |
| `kafka-patterns` | Kafka partition sizing, consumer lag, exactly-once, message ordering |
| `backend-go-samber-hot` | In-memory caching with samber/hot (LRU, LFU, TinyLFU, TTL, Prometheus) |

### Error Handling & Safety

| Skill | When to use |
|---|---|
| `backend-go-error-handling` | Error creation, wrapping (%w), sentinel errors, AppError HTTP boundary, single handling rule |
| `backend-go-safety` | Nil safety, slice/map pitfalls, numeric conversions, defer in loops, defensive copying |
| `backend-go-security` | Injection prevention, cryptography, TLS, secrets management, threat modeling |

### Concurrency & Performance

| Skill | When to use |
|---|---|
| `backend-go-concurrency` | Goroutines, channels, sync primitives, errgroup, singleflight, worker pools |
| `backend-go-context` | context.Context propagation, cancellation, timeouts, deadlines, tracing |
| `backend-go-performance` | Optimization patterns after profiling — allocation reduction, pooling, caching |
| `backend-go-benchmark` | Benchmarking, pprof profiling, benchstat comparison, CI regression detection |

### Testing

| Skill | When to use |
|---|---|
| `backend-go-testing` | Table-driven tests, integration tests, fuzzing, coverage, goleak, CI setup |
| `backend-go-stretchr-testify` | testify assert/require/mock/suite, argument matchers, call verification |

### Observability & Debugging

| Skill | When to use |
|---|---|
| `backend-go-observability` | Structured logging (prep-go-log), Prometheus metrics, OTel tracing, Grafana |
| `backend-go-troubleshooting` | Systematic debugging — pprof, Delve, race detection, GODEBUG, production issues |

### Tooling & CI/CD

| Skill | When to use |
|---|---|
| `backend-go-linter` | golangci-lint configuration, nolint directives, lint output interpretation |
| `backend-go-continuous-integration` | GitHub Actions workflows, SAST, security scanning, GoReleaser, Dependabot |
| `backend-go-dependency-management` | go.mod, versioning, vulnerability scanning, Renovate, dependency analysis |
| `backend-go-modernize` | Modern Go features (1.21–1.26), deprecated packages, tooling upgrades |

### Communication & Infrastructure

| Skill | When to use |
|---|---|
| `backend-go-grpc` | gRPC servers/clients, protobuf organization, interceptors, streaming RPCs |
| `backend-go-cli` | CLI development with Cobra/Viper, flags, configuration, signal handling |
| `backend-go-documentation` | Godoc, README, CHANGELOG, Example tests, API docs, llms.txt |

### Reference

| Skill | When to use |
|---|---|
| `backend-go-popular-libraries` | Vetted production-ready library recommendations |
| `backend-go-stay-updated` | Go news, communities, learning resources, people to follow |

### Project-Specific

| Skill | When to use |
|---|---|
| `myvocap-backend` | MyVocab feature module template, Deps/Provide DI, error flow, sqlc repo, cache-aside, response envelope, middleware wiring. **Overrides generic skills when they conflict.** |
