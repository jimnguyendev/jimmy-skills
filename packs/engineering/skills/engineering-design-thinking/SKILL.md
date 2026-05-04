---
name: engineering-design-thinking
description: "Design thinking process for large features. Use BEFORE implementation to frame the problem, evaluate constraints, decide architecture, and route to the right implementation skills. Invoke explicitly with /design or when starting any feature that spans multiple files, services, or concerns."
user-invocable: true
license: MIT
compatibility: "Designed for Claude Code or similar AI coding agents. Stack-agnostic — works for backend, frontend, or full-stack features."
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent WebFetch WebSearch AskUserQuestion
---

thinking: ultrathink

# Design Thinking Process

Use this skill before writing code for any feature that is non-trivial.

**Non-trivial** = touches multiple files, introduces a new concept, changes data flow, adds a dependency, or requires a migration.

## Core Principle

Programming is thinking, not typing.
AI can produce 10,000 lines per day. The engineering problem is deciding which lines matter.

This skill enforces that decision process through five gates.
No implementation begins until all gates are satisfied.

## Modes

### Think mode (default)

Walk through all five gates. Produce a design brief at the end.
Refuse to write implementation code until the brief is accepted.

### Review mode

Given an existing design or PR, evaluate it against the five gates.
Flag any gate that was skipped or insufficiently addressed.

### Scope mode

Given a feature request, determine scope boundaries and identify which gates require the most attention. Output a prioritized checklist.

---

## The Five Gates

```text
  "Build feature X"
        │
        ▼
  ┌─ GATE 1 ─┐     What is the real problem?
  │  Problem  │     Requirements? Constraints? Who benefits?
  └─────┬─────┘
        ▼
  ┌─ GATE 2 ─┐     What exists today?
  │  Context  │     Codebase state? Dependencies? Team constraints?
  └─────┬─────┘
        ▼
  ┌─ GATE 3 ─┐     What are the options?
  │  Options  │     Solution ideality? Trade-off resolution?
  └─────┬─────┘
        ▼
  ┌─ GATE 4 ─┐     How should it be structured?
  │  Design   │     Boundaries? Data flow? Failure modes?
  └─────┬─────┘
        ▼
  ┌─ GATE 5 ─┐     What skills guide implementation?
  │  Routing  │     Which skills, in what order?
  └───────────┘
```

### Gate 1: Problem Framing

Do not start with "how." Start with "what" and "why."

| Question | Must answer |
|----------|-------------|
| What is the user-facing outcome? | One sentence, no jargon |
| What are the functional requirements? | List, prioritized |
| What are the non-functional requirements? | Latency, throughput, availability, security |
| What assumptions are we making? | Label each as verified or unverified |
| What is NOT in scope? | Explicit exclusions prevent creep |

**Anti-patterns:**

- Jumping to solution before understanding the problem
- Anchoring on the first framing without challenging it
- Treating vague requirements as constraints ("it should be fast" is not a requirement)

### Gate 2: Context Analysis

Understand the existing system before proposing changes.

| Question | Must answer |
|----------|-------------|
| What code exists in this area? | Files, packages, data flow |
| What are the existing dependencies? | Internal and external |
| What is the team's expertise? | Relevant tech stack experience |
| What is the deployment environment? | Infra constraints, scaling model |
| What is the timeline? | Hard deadline vs. flexible |
| What tech debt is in the path? | Must fix vs. can work around |

**Actions:**

1. Read the relevant code before proposing anything
2. Check git history for recent changes in the area
3. Identify existing patterns the codebase uses
4. Note any friction points that will affect implementation

### Gate 3: Solution Evaluation

Evaluate options by Solution Ideality, not by elegance.

```
Solution Ideality = Benefits / (Resources Required + Harmful Effects)
```

- **Resources**: time, cost, people, infrastructure
- **Harmful effects**: known AND unknown (coupling, operational burden, maintenance cost)
- A solution that looks clean but introduces hidden coupling scores low

**Process:**

1. List at least 2 distinct approaches (avoid single-option bias)
2. For each approach, score: benefits, resources, harmful effects
3. Identify trade-offs — then ask: is this a real trade-off or coupling in disguise?
4. If it is coupling, resolve it (separate concerns) instead of picking a side
5. Choose the approach with the highest ideality ratio

**Anti-patterns:**

- Evaluating only one option
- Choosing the most "interesting" or "modern" approach
- Over-engineering for hypothetical future requirements
- Under-engineering because "we can refactor later"

### Gate 4: Architecture Decision

Structure the solution before writing it.

| Decision | Criteria |
|----------|----------|
| Feature-first or layer-first? | Default: feature-first unless strong reason |
| How many packages/modules? | Start with fewer, split only when pain appears |
| What are the boundaries? | Each unit should own one business capability |
| What is the dependency direction? | Must form a DAG — no cycles |
| What is the data flow? | Request → processing → persistence → response |
| What are the failure modes? | Network, timeout, partial failure, data inconsistency |
| What is the contention model? | Low contention vs. high contention → different patterns |

**Output:** A brief architecture sketch — not a diagram tool, just text:

```
Feature: [name]
Packages: [list with one-line responsibility each]
Dependencies: [A → B → C, no cycles]
Data flow: [request path, happy and error]
Key decisions: [1-3 non-obvious choices with rationale]
```

### Gate 5: Skill Routing

Map the implementation to the right skills in the right order.

**Routing table:**

| Problem type | Start with | Then |
|--------------|-----------|------|
| Backend architecture | `jimmy-skills@backend-core` | Language-specific skill |
| Go implementation | `jimmy-skills@backend-go-project-layout` | Relevant `backend-go-*` skill |
| MyVocab Go feature | `jimmy-skills@myvocap-backend` | Feature module template |
| API design | `jimmy-skills@engineering-rest-api-design` | Backend skill for implementation |
| Performance concern | `jimmy-skills@engineering-perf-optimization-process` | Language-specific perf skill |
| Full-stack feature | This skill → `backend-core` | Implementation skills |

**Output:** An ordered skill sequence for the implementation phase.

---

## Design Brief Template

After all five gates are satisfied, produce this brief:

```
## Design Brief: [Feature Name]

### Problem
[One paragraph from Gate 1]

### Context
[Key findings from Gate 2]

### Chosen Approach
[Selected option from Gate 3 with ideality rationale]

### Architecture
[Sketch from Gate 4]

### Implementation Plan
[Ordered skill sequence from Gate 5]
[Estimated scope: files to create/modify]

### Risks & Mitigations
[From failure mode analysis in Gate 4]

### Out of Scope
[Explicit exclusions from Gate 1]
```

**Do not begin implementation until the user accepts the brief.**

---

## When to Use This Skill

| Trigger | Action |
|---------|--------|
| User says "build feature X" or "implement X" | Run full five gates |
| User says "I need to add..." with scope > 1 file | Run full five gates |
| User says "/design" | Run full five gates |
| User says "review this design" | Run in review mode |
| User says "what's the scope of X?" | Run in scope mode |
| User asks a single implementation question | Skip — route directly to implementation skill |

## Recommended Hooks

For teams that want design thinking enforced automatically, add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Write",
        "command": "echo 'Creating new file — have you run /design for this feature?'"
      }
    ]
  }
}
```

This provides a gentle reminder when creating new files, without blocking single-file edits.

## Cross-References

- `jimmy-skills@backend-core` — Backend-specific design principles
- `jimmy-skills@myvocap-backend` — MyVocab project-specific feature module patterns
- `jimmy-skills@engineering-perf-optimization-process` — Performance-specific gate process
- `jimmy-skills@engineering-rest-api-design` — API contract design
