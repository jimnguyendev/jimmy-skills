# jimmy-skills

An opinionated skill kit that teaches AI agents how experienced engineers build software — not just what code to write, but when to write it, when to stop, and how to prove it works.

## Install

### Any AI agent (Gemini, Codex, Cursor, Copilot, Windsurf, Cline, ...)

```bash
npx skills add jimnguyendev/jimmy-skills --skill '*' --agent codex
```

Recommended: always pass `--agent`, and usually `--skill '*'` if you want the full pack for that agent.

Avoid running bare `npx skills add jimnguyendev/jimmy-skills` unless you intentionally want the installer to inspect every detected/supported agent. Users have reported that the bare command can create config directories for agents they do not use yet, such as `.kilo` or `.gemini`, which then need manual cleanup.

```bash
# Install all skills to one agent only
npx skills add jimnguyendev/jimmy-skills --skill '*' --agent codex
npx skills add jimnguyendev/jimmy-skills --skill '*' --agent claude-code
npx skills add jimnguyendev/jimmy-skills --skill '*' --agent gemini-cli

# Install all skills to multiple specific agents
npx skills add jimnguyendev/jimmy-skills --skill '*' --agent claude-code --agent codex
npx skills add jimnguyendev/jimmy-skills --skill '*' -a gemini-cli -a cursor

# Interactive skill selection, but still scoped to one agent
npx skills add jimnguyendev/jimmy-skills --agent codex

# Install specific skills to specific agents
npx skills add jimnguyendev/jimmy-skills --skill backend-go-performance --agent claude-code

# Install all skills to all detected agents (skip prompts, broad)
npx skills add jimnguyendev/jimmy-skills --all

# List available skills without installing
npx skills add jimnguyendev/jimmy-skills --list
```

Agent names: `claude-code`, `codex`, `gemini-cli`, `cursor`, `github-copilot`, `windsurf`, `cline`, `amp`, and [40+ more](https://github.com/vercel-labs/skills). Powered by [vercel-labs/skills](https://github.com/vercel-labs/skills). Also works with `bunx` or `pnpx`.

### Claude Code

```bash
# Step 1: Add the marketplace
/plugin marketplace add jimnguyendev/jimmy-skills

# Step 2: Install the plugins you need
/plugin install backend@jimmy-skills          # architecture + Go (31 skills)
/plugin install engineering@jimmy-skills      # design thinking + API + perf (3 skills)
/plugin install data-engineer@jimmy-skills    # data platform + tooling + trust + leadership (8 skills)
/plugin install outcomes@jimmy-skills         # outcome thinking + planning + transformation (4 skills)
```

After installing, run `/reload-plugins` to activate.

Skills are namespaced by plugin: `/backend:backend-go-performance`, `/engineering:engineering-design-thinking`, `/data-engineer:data-engineering`, `/outcomes:outcome-thinking`, etc.

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

> Programming is thinking, not typing. Structure serves clarity, not paradigm.

### 1. Organize around business capabilities

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

### 2. Start with fewer packages

- One package is often enough at the beginning.
- Split when pain appears, not before.

### 3. Keep names short

- File names should not repeat the package name.
- Types should not repeat the package name.

### 4. Keep types near usage

- Request/response types stay near the transport layer.
- Persistence-only types stay near the repository.

### 5. Keep dependency direction one-way

Package imports must form a DAG. Circular dependencies indicate a boundary problem.

### 6: Constrain before you optimize

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

Nothing goes in just because the AI sounded confident.

## Available plugins

| Plugin | Skills | Description |
|--------|--------|-------------|
| `backend` | 31 | Backend architecture + Go implementation |
| `engineering` | 3 | Design thinking, API design, perf optimization |
| `data-engineer` | 8 | Data platform, tooling, quality, observability, leadership |
| `outcomes` | 4 | Outcome thinking, planning, operating model, transformation |

### Skill list

```text
backend (31 skills)
├── backend-core                           Shared architectural rules
├── backend-go-benchmark                   Benchmarking methodology
├── backend-go-cli                         CLI application patterns
├── backend-go-code-style                  Readability, naming, boundaries
├── backend-go-concurrency                 Goroutines, channels, atomics
├── backend-go-context                     Cancellation, timeouts
├── backend-go-continuous-integration      CI pipelines
├── backend-go-data-structures             Slices, maps, custom types
├── backend-go-database                    Queries, pooling, N+1, migrations
├── backend-go-dependency-management       go.mod, versioning
├── backend-go-design-patterns             Factory, strategy, DI
├── backend-go-documentation               Godoc conventions
├── backend-go-error-handling              Wrapping, sentinel errors
├── backend-go-grpc                        Protobuf, interceptors
├── backend-go-linter                      golangci-lint
├── backend-go-modernize                   Latest Go idioms
├── backend-go-naming                      Package, type, function names
├── backend-go-observability               Logging, metrics, tracing
├── backend-go-performance                 Profiling, caching, hot path
├── backend-go-popular-libraries           Ecosystem overview
├── backend-go-project-layout              Feature-first layout
├── backend-go-safety                      Race conditions, nil safety
├── backend-go-samber-hot                  Hot-reloading
├── backend-go-security                    Input validation, auth, OWASP
├── backend-go-stay-updated                Go version updates
├── backend-go-stretchr-testify            Assertions, mocks, suites
├── backend-go-structs-interfaces          Composition, embedding
├── backend-go-testing                     Test patterns, integration tests
├── backend-go-troubleshooting             Debugging, profiling
├── kafka-patterns                         Kafka partitioning, lag, delivery guarantees
└── myvocap-backend                        Project-specific backend conventions

engineering (3 skills)
├── engineering-design-thinking            /design — five gates before implementation
├── engineering-rest-api-design            API contracts, versioning
└── engineering-perf-optimization-process  Perf gates, escalation ladder

data-engineer (8 skills)
├── data-engineering                       Warehouse, marts, semantic metrics
├── data-stack-delivery                    Airflow, Snowflake, dbt, Spark, Kafka
├── data-architecture-strategy             RDW vs lakehouse vs mesh
├── data-pipeline-reliability              Retries, idempotency, backfills
├── data-quality                           Contracts, reconciliation, publish gates
├── data-observability                     Freshness, lag, stale dashboards
├── data-program-leadership                Roadmaps, ownership, stakeholder alignment
└── data-value-patterns                    Enrichment, aggregation, value framing

outcomes (4 skills)
├── outcome-thinking                       Outcomes vs outputs vs impact
├── outcomes-based-planning                Journey maps, hypotheses, experiments
├── organizing-for-outcomes                Team topology, intake, trust repair
└── outcomes-driven-transformation         Internal adoption and behavior change
```

### Routing guide

```text
"Build a new payment service"              → /engineering:engineering-design-thinking
"How should I structure this service?"     → backend-core (auto-routed)
"How do I organize Go packages?"           → backend-go-project-layout (auto-routed)
"This endpoint is slow"                    → /engineering:engineering-perf-optimization-process
"Design a REST API"                        → engineering-rest-api-design (auto-routed)
"Review this concurrent code"              → backend-go-concurrency (auto-routed)
"How should we structure our data stack?"  → /data-engineer:data-engineering
"How do Airflow, Snowflake, dbt, and Kafka fit together?" → /data-engineer:data-stack-delivery
"Our dashboards are stale"                 → data-observability (auto-routed)
"Turn this roadmap into outcomes"          → /outcomes:outcome-thinking
```

## Repo layout

```text
.claude-plugin/
  marketplace.json                   Marketplace catalog (4 plugins)
packs/                               Each pack = one installable plugin
  backend/
    .claude-plugin/plugin.json       Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/
      backend-core/SKILL.md
      backend-go-*/SKILL.md          28 Go skills + references/
      myvocap-backend/SKILL.md       Project-specific patterns + references/
  engineering/
    .claude-plugin/plugin.json       Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/
      engineering-design-thinking/SKILL.md
      engineering-rest-api-design/SKILL.md
      engineering-perf-optimization-process/SKILL.md
  data-engineer/
    .claude-plugin/plugin.json       Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/
      data-engineering/SKILL.md
      data-quality/SKILL.md
      data-program-leadership/SKILL.md
      ...                            4 more data skills
  outcomes/
    .claude-plugin/plugin.json       Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/
      outcome-thinking/SKILL.md
      outcomes-based-planning/SKILL.md
      organizing-for-outcomes/SKILL.md
      outcomes-driven-transformation/SKILL.md
```

## How it works

```text
/plugin marketplace add jimnguyendev/jimmy-skills
  └─ clones repo, reads .claude-plugin/marketplace.json
     └─ registers 4 plugins: backend, engineering, data-engineer, outcomes

/plugin install backend@jimmy-skills
  └─ copies packs/backend/ to ~/.claude/plugins/cache
     └─ plugin.json says skills: ./skills/
        └─ Claude scans skills/ → 31 skills available
```

## Team setup

Add to your project's `.claude/settings.json` so team members get prompted automatically:

```json
{
  "extraKnownMarketplaces": {
    "jimmy-skills": {
      "source": {
        "source": "github",
        "repo": "jimnguyendev/jimmy-skills"
      }
    }
  },
  "enabledPlugins": {
    "backend@jimmy-skills": true
  }
}
```

## Attribution

Go backend skills were originally derived from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang). This repo has since diverged significantly — restructured around feature-first architecture, rewritten to match Jim Nguyen's conventions, and extended with new skills.

## Notes

- Cross-skill references use `jimmy-skills@<skill-name>`.
- Plugin skills are namespaced: `/backend:skill-name`, `/engineering:skill-name`, `/data-engineer:skill-name`, `/outcomes:skill-name`.
- Upstream references are kept only where they still add concrete value.
