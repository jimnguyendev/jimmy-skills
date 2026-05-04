# Quality Gates for Performance Work

Performance optimization generates code faster than any other type of engineering work. Without gates, the codebase grows faster than your ability to understand, secure, and operate it.

## The Problem

AI can produce 10K lines of optimized code per day. Without constraints:

- Singleflight gets added to a 200 RPS service that has never had a stampede
- L1+L2 cache gets added when the database query takes 3ms and is called once per request
- Lock-free atomic patterns replace a mutex that is held for 50ns with zero contention
- Zero-serialization wrappers are built for an endpoint that handles 10 req/s

Each of these "optimizations" adds complexity that must be understood, tested, monitored, and maintained. The cost compounds with every layer.

## Gate Structure

### Gate: CI Pipeline

Every optimization PR must pass CI before merge. No exceptions.

**Minimum checks:**

- Formatting and linting pass
- Type checking passes
- Existing test suite passes (optimization must not break behavior)
- New benchmarks are included and pass
- No merge if CI is red

**AI is useful here:** it can write tests, fix lint errors, and refactor safely. The gate keeps it honest.

### Gate: Load Test Proof

Every optimization PR must include before/after load test results.

**Requirements:**

- Dataset matches production cardinality and payload sizes
- Load test runs for sufficient duration (minimum 60s sustained, not burst)
- Results include latency histogram (p50, p95, p99), throughput, error rate
- Flamegraph comparison for hot-path changes
- Results are committed to the repo or linked in the PR

**"Works on my machine" is not proof.** "Load test shows p99 dropped from 240ms to 80ms at 2,500 RPS sustained for 5 minutes" is proof.

### Gate: Rollback Mechanism

Every optimization must be independently disableable.

**Requirements:**

- Feature flag for each optimization (L1 cache, singleflight, zero-ser, etc.)
- Documented behavior when flag is off
- Alert rules that trigger when the optimization causes regression
- Circuit breaker for new external dependencies (Redis, message queue)

**Why independent:** If you ship L1 cache + singleflight + zero-ser in one release and p99 regresses, you need to know which one caused it. Feature flags let you disable each independently without rolling back the entire release.

### Gate: Observability Before Optimization

You cannot optimize what you cannot measure. You cannot detect regression without monitoring.

**Before any optimization work begins, verify:**

- Latency histogram exists for hot-path endpoints (p50, p95, p99)
- Error rate is tracked
- CPU and memory gauges exist
- Dependency latency is tracked (DB, Redis, external APIs)

**After each optimization, add:**

- Cache hit rate (L1 and Redis separately)
- Singleflight dedup rate (if applicable)
- Timeout count, retry count, shed count
- Dashboard showing before/after for the optimized metric

### Gate: Security Automation

Optimization code is still code. It must pass the same security checks.

**Minimum checks:**

- Dependency scanning (no known vulnerabilities in new dependencies)
- Secret scanning (no credentials in cache keys, config, or test data)
- SAST rules: no string-built queries, no unsanitized input in cache keys
- No "temporary" bypasses that become permanent (admin endpoints, debug flags in production)

## Integration Testing for Critical Paths

Do not try to test everything. Pick the paths that hurt most when they break:

- Auth and permissions (cache must not serve user A's data to user B)
- Data integrity (cache invalidation must propagate correctly on write)
- Fallback behavior (what happens when Redis is down? when L1 is full?)
- Timeout behavior (does the 50ms query timeout actually return 503, or does it hang?)

**Integration tests catch spaghetti behavior even when unit tests pass.** A cache that serves stale auth tokens is worse than a slow cache.

## The Fundamental Tension

AI optimizes for "task done." Engineering optimizes for "still works next month, under load, with new features, and with other people touching it."

If "task done" wins by default, temporary fixes become permanent architecture. The gates ensure every optimization earns its place by proving it is needed, measured, reversible, and monitored.
