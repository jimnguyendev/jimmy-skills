# jimmy-skills

An opinionated skill kit that teaches AI agents how experienced engineers build software — not just what code to write, but when to write it, when to stop, and how to prove it works.

## Install

### Claude Code (per pack)

Add the marketplace once, then install each pack you need:

```bash
# Step 1: add the marketplace
/plugin marketplace add jimnguyendev/jimmy-skills

# Step 2: install one or more packs
/plugin install backend@jimmy-skills              # architecture + Go (31 skills)
/plugin install engineering@jimmy-skills          # design thinking + API + perf (3 skills)
/plugin install data-engineer@jimmy-skills        # data platform + quality + product analytics + leadership (11 skills)
/plugin install outcomes@jimmy-skills             # outcomes + planning + transformation (4 skills)
/plugin install prompt-engineering@jimmy-skills   # prompt design + outputs + agents + domains (26 skills)
```

After installing, run `/reload-plugins` to activate.

Skills are namespaced by plugin: `/backend:backend-go-performance`, `/engineering:engineering-design-thinking`, `/data-engineer:data-engineering`, `/outcomes:outcome-thinking`, `/prompt-engineering:prompt-engineering-core`, etc.

### Any AI agent (Gemini, Codex, Cursor, Copilot, Windsurf, Cline, …) via npx

The `npx skills add` installer (powered by [vercel-labs/skills](https://github.com/vercel-labs/skills)) writes skill files into your agent's config directory. Pack selection requires **exact skill names** — the installer does literal-name matching, not glob patterns. Copy the block for the pack you want:

```bash
# backend (31 skills)
npx skills add jimnguyendev/jimmy-skills --agent codex \
  --skill backend-core \
  --skill backend-go-benchmark --skill backend-go-cli --skill backend-go-code-style \
  --skill backend-go-concurrency --skill backend-go-context --skill backend-go-continuous-integration \
  --skill backend-go-data-structures --skill backend-go-database --skill backend-go-dependency-management \
  --skill backend-go-design-patterns --skill backend-go-documentation --skill backend-go-error-handling \
  --skill backend-go-grpc --skill backend-go-linter --skill backend-go-modernize \
  --skill backend-go-naming --skill backend-go-observability --skill backend-go-performance \
  --skill backend-go-popular-libraries --skill backend-go-project-layout --skill backend-go-safety \
  --skill backend-go-samber-hot --skill backend-go-security --skill backend-go-stay-updated \
  --skill backend-go-stretchr-testify --skill backend-go-structs-interfaces --skill backend-go-testing \
  --skill backend-go-troubleshooting \
  --skill kafka-patterns --skill myvocap-backend

# engineering (3 skills)
npx skills add jimnguyendev/jimmy-skills --agent codex \
  --skill engineering-design-thinking \
  --skill engineering-rest-api-design \
  --skill engineering-perf-optimization-process

# data-engineer (11 skills)
npx skills add jimnguyendev/jimmy-skills --agent codex \
  --skill data-engineering --skill data-stack-delivery --skill data-architecture-strategy \
  --skill product-metrics-design --skill retention-engagement-analysis --skill experimentation-analytics \
  --skill data-pipeline-reliability --skill data-quality --skill data-observability \
  --skill data-program-leadership --skill data-value-patterns

# outcomes (4 skills)
npx skills add jimnguyendev/jimmy-skills --agent codex \
  --skill outcome-thinking --skill outcomes-based-planning \
  --skill organizing-for-outcomes --skill outcomes-driven-transformation

# prompt-engineering (26 skills)
npx skills add jimnguyendev/jimmy-skills --agent codex \
  --skill prompt-engineering-core --skill prompt-engineering-system-prompt \
  --skill prompt-engineering-role --skill prompt-engineering-pitfalls \
  --skill prompt-engineering-few-shot --skill prompt-engineering-reasoning \
  --skill prompt-engineering-refine --skill prompt-engineering-edge-cases \
  --skill prompt-engineering-output-structured --skill prompt-engineering-output-json \
  --skill prompt-engineering-output-xml --skill prompt-engineering-output-yaml \
  --skill prompt-engineering-chain --skill prompt-engineering-context \
  --skill prompt-engineering-cost --skill prompt-engineering-eval \
  --skill prompt-engineering-agent --skill prompt-engineering-multimodal \
  --skill prompt-engineering-domain-coding --skill prompt-engineering-domain-writing \
  --skill prompt-engineering-domain-education --skill prompt-engineering-domain-business \
  --skill prompt-engineering-domain-creative --skill prompt-engineering-domain-research \
  --skill prompt-engineering-domain-support --skill prompt-engineering-domain-devops
```

Combine packs by stacking `--skill` flags from multiple blocks. Combine agents by stacking `--agent` (or `-a`):

```bash
# One pack to multiple agents — stack -a flags
npx skills add jimnguyendev/jimmy-skills \
  -a claude-code -a codex -a cursor \
  --skill engineering-design-thinking \
  --skill engineering-rest-api-design \
  --skill engineering-perf-optimization-process

# Interactive skill picker, scoped to one agent
npx skills add jimnguyendev/jimmy-skills --agent codex

# Install ALL 72 skills to one agent
npx skills add jimnguyendev/jimmy-skills --skill '*' --agent codex

# List available skills without installing
npx skills add jimnguyendev/jimmy-skills --list
```

> **Always pass `--agent`.** Running bare `npx skills add jimnguyendev/jimmy-skills` causes the installer to inspect every detected/supported agent and may create config directories for agents you don't use (`.kilo`, `.gemini`, etc.) that then need manual cleanup.

Agent names: `claude-code`, `codex`, `gemini-cli`, `cursor`, `github-copilot`, `windsurf`, `cline`, `amp`, and [40+ more](https://github.com/vercel-labs/skills). Also works with `bunx` or `pnpx`.

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
| `data-engineer` | 11 | Data platform, tooling, quality, observability, product analytics, leadership |
| `outcomes` | 4 | Outcome thinking, planning, operating model, transformation |
| `prompt-engineering` | 26 | Prompt frame, role/system design, structured outputs, reasoning, chains, context, cost, eval, agents, multimodal, 8 domain templates |

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

data-engineer (11 skills)
├── data-engineering                       Warehouse, marts, semantic metrics
├── data-stack-delivery                    Airflow, Snowflake, dbt, Spark, Kafka
├── data-architecture-strategy             RDW vs lakehouse vs mesh
├── data-pipeline-reliability              Retries, idempotency, backfills
├── data-quality                           Contracts, reconciliation, publish gates
├── data-observability                     Freshness, lag, stale dashboards
├── data-program-leadership                Roadmaps, ownership, stakeholder alignment
├── data-value-patterns                    Enrichment, aggregation, value framing
├── product-metrics-design                 North Star, guardrails, anti-vanity
├── retention-engagement-analysis          Cohorts, state machines, aha moments
└── experimentation-analytics              A/B design, power, exposure, capacity

outcomes (4 skills)
├── outcome-thinking                       Outcomes vs outputs vs impact
├── outcomes-based-planning                Journey maps, hypotheses, experiments
├── organizing-for-outcomes                Team topology, intake, trust repair
└── outcomes-driven-transformation         Internal adoption and behavior change

prompt-engineering (26 skills)
├── prompt-engineering-core                Prompt Frame, Output Contract, Cost Profile, routing
├── prompt-engineering-system-prompt       Persistent identity (5-part + layered architecture)
├── prompt-engineering-role                Role stack, compound/situational/perspective
├── prompt-engineering-pitfalls            9 anti-patterns + ship checklist
├── prompt-engineering-few-shot            2–5 rule, diversity, negative examples, ordering
├── prompt-engineering-reasoning           Chain-of-thought, BREAK, scratchpad, self-consistency
├── prompt-engineering-refine              Write→Test→Analyze→Improve, single-variable iteration
├── prompt-engineering-edge-cases          Input/Domain/Adversarial, prompt-injection defense
├── prompt-engineering-output-structured   Markdown lists/tables/headers/emphasis
├── prompt-engineering-output-json         Schema-first JSON, null-handling, parsers
├── prompt-engineering-output-xml          Claude-native tags, sandboxing, mixed content
├── prompt-engineering-output-yaml         k8s/Compose/CI; Norway problem; safe_load
├── prompt-engineering-chain               Sequential/parallel/conditional/iterative chains
├── prompt-engineering-context             RAG, embeddings, summarization, tools, MCP
├── prompt-engineering-cost                Routing, prefix cache, compression, batch, streaming
├── prompt-engineering-eval                Golden fixtures, scoring, CI gating
├── prompt-engineering-agent               Plan→Execute→Observe→Adapt, recovery, budgets
├── prompt-engineering-multimodal          Image/audio/video; six-slot image-gen; OCR-trust
├── prompt-engineering-domain-coding       PR review, debug, gen, arch, tests, refactor
├── prompt-engineering-domain-writing      Blog, marketing, brand-voice, three-phase editing
├── prompt-engineering-domain-education    Adaptive tutor, Socratic, lesson, quiz, paths
├── prompt-engineering-domain-business     Email, SWOT, agenda, OKR, plan, SOP
├── prompt-engineering-domain-creative     Image-gen, three-act, character, song, brainstorm
├── prompt-engineering-domain-research     Paper summary, synthesis, PESTLE, 5 Whys, CRAAP
├── prompt-engineering-domain-support      Triage, KB-grounded reply, escalation, KB authoring
└── prompt-engineering-domain-devops       Incident, postmortem, runbook, IaC review, comms
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
"What should our North Star and guardrail metrics be?" → /data-engineer:product-metrics-design
"Why do users churn and what is our aha moment?" → /data-engineer:retention-engagement-analysis
"Design this A/B test properly"             → /data-engineer:experimentation-analytics
"Our dashboards are stale"                 → data-observability (auto-routed)
"Turn this roadmap into outcomes"          → /outcomes:outcome-thinking
"How should I structure this prompt?"      → /prompt-engineering:prompt-engineering-core
"Design a system prompt for an agent"      → prompt-engineering-system-prompt (auto-routed)
"Make this prompt return JSON reliably"    → prompt-engineering-output-json (auto-routed)
"My LLM costs / latency are too high"      → /prompt-engineering:prompt-engineering-cost
"Build a tool-using agent"                 → /prompt-engineering:prompt-engineering-agent
"Review this PR with an LLM"               → prompt-engineering-domain-coding (auto-routed)
```

## Repo layout

```text
.claude-plugin/
  marketplace.json                   Marketplace catalog (5 plugins)
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
  prompt-engineering/
    .claude-plugin/plugin.json       Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/
      prompt-engineering-core/SKILL.md
      prompt-engineering-system-prompt/SKILL.md
      prompt-engineering-role/SKILL.md
      ...                            6 foundations + 6 techniques
      prompt-engineering-chain/SKILL.md
      prompt-engineering-agent/SKILL.md
      ...                            6 production-runtime skills
      prompt-engineering-domain-coding/SKILL.md
      prompt-engineering-domain-writing/SKILL.md
      ...                            8 domain templates
```

## How it works

```text
/plugin marketplace add jimnguyendev/jimmy-skills
  └─ clones repo, reads .claude-plugin/marketplace.json
     └─ registers 5 plugins: backend, engineering, data-engineer, outcomes, prompt-engineering

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
- Plugin skills are namespaced: `/backend:skill-name`, `/engineering:skill-name`, `/data-engineer:skill-name`, `/outcomes:skill-name`, `/prompt-engineering:skill-name`.
- Upstream references are kept only where they still add concrete value.
