# jimmy-skills

`jimmy-skills` is a stack-first skill kit for backend, frontend, and engineering work.

This repo is opinionated. For backend APIs, it prefers:

- feature-first packages over technical-layer folders
- simple boundaries before abstraction
- types close to where they are used
- short, non-redundant names
- one-way dependencies and consumer-side interfaces

It is not trying to preserve the structure of `samber/cc-skills-golang`. Some skills still reuse or reference upstream material where useful, especially library-specific backend skills, but the repo direction is now driven by Jim Nguyen's architecture conventions.

## Core Philosophy

### 1. Organize around business capabilities

If you are building a Go API, the default is to group code by feature, not by role.

Prefer this:

```text
internal/
  users/
    handler.go
    service.go
    repository.go
    routes.go
    types.go
  invoices/
  posts/
```

Over this:

```text
internal/
  handlers/
  services/
  repository/
  models/
```

The goal is locality. If you change one business capability, you should mostly work in one place. Keep those role-based files only when they are earning their keep; one or two files in a feature package is often enough early on.

### 2. Start with fewer packages

Do not design five layers before the code needs them.

- one package is often enough at the beginning
- split when pain appears, not before
- if one small change forces edits across many packages, the boundaries are wrong

### 3. Keep names short

Avoid naming noise.

- file names should not repeat the package name
- types should not repeat the package name
- methods should not repeat the receiver type name

If the package already says `users`, prefer `Service.Create`, not `UserService.CreateUser`.

### 4. Keep types near usage

Avoid giant shared buckets like `models.go`.

- request and response types should stay near the transport layer that owns them
- persistence-only types should stay near the repository that uses them
- feature-shared types can live in that feature's `types.go`
- extract shared packages only when the sharing is real

### 5. Keep dependency direction one-way

Package imports must form a DAG.

If two features need each other:

1. move behavior to the package that owns the concern
2. merge the packages if they are really one unit
3. define a small interface in the consumer package and inject the concrete dependency from wiring code

Import cycles are treated as a boundary problem, not as a tooling annoyance.

### 6. Constrain before you optimize

AI can produce thousands of lines of optimized code per day. Without engineering constraints, that code creates systems harder to understand, debug, and operate than the "slow" version it replaced.

The gap between a casual request ("make it high performance") and an engineering specification (concrete latency targets, CPU budgets, profiling proof, rollback plans) is not a typing problem. It is a judgment problem. Skills in this repo encode that judgment.

**Rules:**

- No optimization without measurable targets. "Make it faster" must become "p95 < 80ms at 2,500 RPS."
- No optimization without profiling proof. Intuition about bottlenecks is wrong ~80% of the time.
- Every optimization follows an escalation ladder: fix the query, then add cache, then add L1, then add stampede protection. Each step requires metric proof before escalating to the next.
- Every optimization must be independently reversible via feature flags.
- Every optimization PR includes before/after load test numbers and a rollback plan.

### 7. Enforce correctness with quality gates

AI optimizes for "task done." Engineering optimizes for "still works next month, under load, with new features, and with other people touching it."

Without quality gates, the codebase grows faster than your ability to understand, secure, and run it. Nothing goes in just because it "works on my machine" or because the AI sounded confident.

**Minimum gates for every change:**

- CI pipeline on every pull request: formatting, linting, type checking, test suite. No merge if red.
- Integration tests for critical paths: auth, permissions, data integrity, money flows, external API contracts. These catch spaghetti behavior even when unit tests pass.
- Security automation: dependency scanning, secret scanning, SAST. No string-built SQL, no unsanitized HTML, no logging secrets, no "temporary" admin bypasses. Boring rules scale better than heroic review.
- Load test proof for performance changes: dataset matching production cardinality, before/after numbers, flamegraph comparison.

The system must force correctness. The easiest way is to make the computer reject bad changes automatically.

## Packs

### Backend

The backend pack follows the philosophy above.

- `backend-core` holds shared backend architectural rules
- `backend-go-project-layout` encodes feature-first layout and import-cycle prevention
- `backend-go-code-style` covers readability, locality, naming noise, and package boundaries
- `backend-go-linter` is explicitly limited to tooling and must not drive architecture decisions
- `backend-go-*` skills cover Go implementation details once the architectural direction is already clear

### Frontend

The frontend pack is intentionally lighter right now.

- `frontend-core` for shared UI architecture
- `frontend-react` for React
- `frontend-vue` for Vue

### Engineering

The engineering pack covers stack-agnostic delivery, review workflows, and constraint-driven process.

- `engineering-rest-api-design` for API contract design
- `engineering-perf-optimization-process` for constraint-driven performance work: five gates (targets, hot path, profiling, escalation ladder, rollback), quality gate enforcement, and validation requirements. Prevents over-engineering by requiring metric proof before applying any optimization pattern.

## Layout

```text
kit.manifest.json
packs/
  backend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/
  frontend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/
  engineering/
    pack.manifest.json
    pack-quickstart.md
```

## Install

```bash
npx skills add https://github.com/jimnguyendev/jimmy-skills --all
```

Or:

```bash
git clone https://github.com/jimnguyendev/jimmy-skills.git ~/.agents/skills/jimmy-skills
```

## Important Files

- [kit.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/kit.manifest.json)
- [packs/backend/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/backend/pack.manifest.json)
- [packs/frontend/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/frontend/pack.manifest.json)
- [packs/engineering/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/engineering/pack.manifest.json)

## Notes

- Cross-skill references use `jimmy-skills@<skill-name>`.
- Legacy publisher metadata and automation are intentionally removed.
- Upstream references are kept only where they still add concrete value.
