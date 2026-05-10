---
name: backend-core
description: "Shared backend design thinking and engineering conventions. Covers solution evaluation, trade-off resolution, service boundaries, API design, persistence, async workflows, release safety, and cross-service contracts. Use when the task is backend but not language-specific yet, or when a request spans architectural decisions, API design, database changes, messaging, rollout, or production behavior across multiple stacks."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.1.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Backend Core

Use this skill before dropping into a language-specific backend skill when the main question is architectural or operational rather than syntax-level.

For large features (multiple files, new concepts, migrations), run `jimmy-skills@engineering-design-thinking` first — it provides the full five-gate process before any implementation begins. The principles below are the backend-specific application of that process.

## Design Thinking

Programming is thinking, not typing.
Every principle below exists to protect that idea.

### 1. Structure serves clarity, not paradigm

Do not pick OOP, procedural, or functional because of ideology.
Pick what makes the code readable after the project stops being small.

- Plain code first. Add methods, interfaces, and abstractions only when
  they solve an actual problem: testing, replacement, or repeated behavior.
- A handler that moves data from request to database does not need
  five abstractions and ten files. That is decoration, not design.
- The test: when this code gets bigger, will it still make sense
  without a lecture? If yes, the structure is right.

### 2. Evaluate by Solution Ideality

Solution Ideality = Benefits / (Resources Required + Harmful Effects)

- Resources: time, cost, people.
- Harmful effects: known and unknown.
- A solution that looks clean but introduces hidden coupling
  or operational burden scores low. Judge by the ratio, not the appearance.
- System design in reality is not system design in interviews.
  Reality has existing systems, constrained budgets, and teams
  that must maintain what you ship.

### 3. Understand the real problem before proposing solutions

- Expected: identify functional requirements, non-functional requirements,
  stakeholders, and timeline.
- Reality: acknowledge the existing system and its constraints
  (team expertise, budget, tech debt, deployment environment).
- Distinguish assumptions from facts. If a claim is not verified,
  label it as an assumption and validate before building on it.
- Do not get anchored by the first framing of a problem.

### 4. Trade-offs are coupling in disguise

- When optimizing A is constrained by B, the root cause is
  coupling between them.
- Do not pick a side of a trade-off. Resolve the coupling:
  separate the concerns so each can evolve independently.
- One-way dependencies and consumer-side interfaces are not
  code-organization rules. They are coupling resolvers.

### 5. Decompose with proof, not instinct

- Start with fewer, larger units. Split only when the split delivers
  measurable value for scalability or development velocity.
- "If splitting does not bring meaningful value, do not split."
- Most bad code comes from trying to look scalable before earning it.
- Be extremely cautious with distributed transactions.
  Each one adds coupling that is hard to remove.

### 6. Classify contention before choosing architecture

- Low contention: requests spread across independent resources
  (wallet transfers between different account pairs).
- High contention: many requests converge on the same resource
  (100k users buying from 10k tickets at the same moment).
- Same "high concurrency" label, fundamentally different solutions.
  Do not apply the same pattern to both.

## Focus Areas

1. API contracts before handler code
2. Zero-downtime schema evolution before migration implementation
3. Idempotency and retries for async flows
4. Authentication and authorization boundaries
5. Observability, rollout, and rollback requirements

## Architecture Defaults

These defaults are the concrete application of the Design Thinking principles above.

For backend APIs, default to **feature-first organization** unless the user explicitly wants another style.

- Group code by business capability, not by technical layer
- Keep one feature mostly in one place
- Package dependencies must form a DAG — this enforces principle 4 (resolve coupling)

If two backend packages need each other, stop and fix the boundary:

1. Move behavior to the package that owns the concern
2. Merge the packages if they are really one unit
3. Define a narrow interface in the consumer package and inject the concrete dependency from wiring code

Circular dependencies indicate a boundary problem, not a tooling inconvenience.

## Routing

- For full design thinking process on large features, use `jimmy-skills@engineering-design-thinking`.
- For Go implementation details, move to the relevant `jimmy-skills@backend-go-*` skill.
- For Go package structure and feature-first layout decisions, prefer `jimmy-skills@backend-go-project-layout`.
- For MyVocab project-specific patterns (feature module, error flow, DI), use `jimmy-skills@myvocap-backend`.
- For REST API contract design, use `jimmy-skills@engineering-rest-api-design`.
- For performance optimization, use `jimmy-skills@engineering-perf-optimization-process`.
