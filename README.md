# jimmy-skills

An opinionated skill kit that teaches AI agents how experienced engineers build software — not just what code to write, but when to write it, when to stop, and how to prove it works.

```bash
/plugin marketplace add jimmy-skills@jimmy-skills
```

## Why this exists

AI agents can produce thousands of lines of code per day. Without constraints, that code creates systems harder to understand, debug, and operate than what it replaced.

The gap between a casual prompt and an engineering specification is not a typing problem. It is a judgment problem.

```
casual:       "please build a high performance news feed app"

experienced:  "p95 < 80ms at 2,500 RPS sustained, CPU < 65%,
               profile before optimizing, feature flag each change,
               load test proves improvement, rollback plan for each PR"
```

This repo encodes that judgment as skills — reusable instruction sets that guide AI agents to make engineering decisions, not just produce output.

## Philosophy

Seven principles drive every skill in this repo.

```text
                    ┌─────────────────────────────────────┐
                    │         HOW WE ORGANIZE              │
                    │                                       │
                    │  1. Feature-first, not layer-first    │
                    │  2. Fewer packages, split when pain   │
                    │  3. Short names, no stuttering        │
                    │  4. Types near where they are used    │
                    │  5. One-way dependencies (DAG)        │
                    │                                       │
                    ├─────────────────────────────────────  │
                    │         HOW WE OPTIMIZE               │
                    │                                       │
                    │  6. Constrain before you optimize     │
                    │                                       │
                    ├─────────────────────────────────────  │
                    │         HOW WE SHIP                   │
                    │                                       │
                    │  7. Enforce correctness with gates    │
                    │                                       │
                    └─────────────────────────────────────┘
```

### 1-5: How we organize code

Group code by business capability, not by technical role.

```text
internal/                          internal/
  users/                             handlers/
    handler.go                       services/
    service.go        PREFER         repository/      AVOID
    repository.go     ────────►      models/
    types.go
  invoices/
  posts/
```

- Start with one package. Split when pain appears, not before.
- Names do not stutter. If the package says `users`, the type is `Service`, not `UserService`.
- Types stay near their usage. No giant shared `models.go`.
- Package imports form a DAG. Cycles mean the boundary is wrong — fix the boundary, not the tooling.

### 6: Constrain before you optimize

No optimization without these:

```text
  "Make it faster"
        │
        ▼
  ┌─ GATE 1 ─┐     What are the hard targets?
  │  Targets  │     p95 < 80ms? 2,500 RPS? CPU < 65%?
  └─────┬─────┘
        ▼
  ┌─ GATE 2 ─┐     Which endpoints are >80% of traffic?
  │ Hot path  │     What latency distribution? Read/write ratio?
  └─────┬─────┘
        ▼
  ┌─ GATE 3 ─┐     What does the profiler say?
  │ Profile   │     CPU bound? I/O bound? Contention?
  └─────┬─────┘
        ▼
  ┌─ GATE 4 ─┐     Escalation ladder: simplest fix first
  │ Solution  │     Fix query → Redis → L1 → singleflight → zero-ser
  └─────┬─────┘     Each step needs metric proof to escalate
        ▼
  ┌─ GATE 5 ─┐     Feature flag? Load test? Rollback plan?
  │ Rollback  │     If you can't roll back independently, don't ship
  └───────────┘
```

### 7: Enforce correctness with quality gates

AI optimizes for "task done." Engineering optimizes for "still works next month."

```text
  Every change must pass:

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   CI green   │  │  Integration │  │   Security    │  │  Load test   │
  │              │  │    tests     │  │  automation   │  │   (if perf)  │
  │ lint, type,  │  │ auth, money, │  │ deps, secrets │  │ before/after │
  │ format, test │  │ permissions  │  │ SAST, no SQL  │  │ flamegraph   │
  │              │  │              │  │ injection     │  │              │
  │  red = stop  │  │ catches      │  │ boring rules  │  │ "works on my │
  │              │  │ spaghetti    │  │ scale better  │  │  machine" is │
  │              │  │              │  │ than heroic   │  │  not proof   │
  │              │  │              │  │ review        │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

Nothing goes in just because the AI sounded confident.

## Skill packs

```text
jimmy-skills
├── backend           30 skills — architecture + Go implementation
│   ├── backend-core                  Shared architectural rules
│   ├── backend-go-project-layout     Feature-first layout, import cycles
│   ├── backend-go-code-style         Readability, naming, boundaries
│   ├── backend-go-performance        Profiling, caching, zero-ser, hot path
│   ├── backend-go-concurrency        Goroutines, channels, atomics, singleflight
│   ├── backend-go-observability      Logging, metrics, tracing, profiling, alerting
│   ├── backend-go-database           Queries, pooling, N+1, migrations
│   ├── backend-go-testing            Test patterns, integration tests, testify
│   └── ... 22 more Go skills
│
├── frontend          3 skills — UI architecture + frameworks
│   ├── frontend-core                 Shared UI architecture
│   ├── frontend-react                React patterns
│   └── frontend-vue                  Vue patterns
│
└── engineering        2 skills — stack-agnostic process
    ├── engineering-rest-api-design            API contracts
    └── engineering-perf-optimization-process  Five gates, escalation ladder,
                                              quality gates, eval test cases
```

### Routing guide

```text
"How should I structure this service?"     → backend-core
"How do I organize Go packages?"           → backend-go-project-layout
"This endpoint is slow"                    → engineering-perf-optimization-process
                                             (then backend-go-performance for Go fixes)
"Add metrics to this handler"              → backend-go-observability
"Design a REST API"                        → engineering-rest-api-design
"Review this concurrent code"              → backend-go-concurrency
```

## Repo layout

```text
kit.manifest.json                    Root manifest (lists all packs)
packs/
  backend/
    pack.manifest.json               Pack manifest (lists all skills)
    pack-quickstart.md
    skills/domain/
      backend-core/SKILL.md          Skill definition + references/
      backend-go-*/SKILL.md
  frontend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/
  engineering/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/
```

## Notes

- Cross-skill references use `jimmy-skills@<skill-name>`.
- Upstream references are kept only where they still add concrete value.
