---
name: prompt-engineering-domain-coding
description: "Ready-to-paste prompt templates for coding tasks: PR review with severity tiers, debugging with expected/actual/error, code generation with full specs, architecture proposal, test generation (AAA), and refactor proposals. Each template fills the Prompt Frame so you can paste, customize the inputs, and ship."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Templates use placeholder language tags; replace per project."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# Domain — Coding & Development

Reusable prompts for the most common code tasks. Each is a Prompt-Frame template: drop in your inputs, ship. Pair with `prompt-engineering-output-xml` (Claude) or `prompt-engineering-output-json` (parseable downstream) when the output is consumed by tooling.

## Template — PR / code review

```text
<role>
You are a senior reviewer who has shipped this stack to production for years.
You review for correctness first, security second, then maintainability.
</role>

<task>
Review the diff for a pull request. Surface issues prioritized by severity.
</task>

<inputs>
<context>{{repo_purpose, conventions_link}}</context>
<diff language="{{lang}}">
{{unified_diff}}
</diff>
</inputs>

<constraints>
- Severity:  🔴 Critical (blocks merge)  🟡 Important  🟢 Nit / suggestion
- Each finding cites file:line and quotes the offending span.
- Do NOT comment on style covered by the linter.
- Do NOT propose unrelated refactors.
</constraints>

<output>
**Verdict:** approve | request_changes | needs_discussion
**Summary:** ≤ 2 sentences for the PR description.

## Findings
- 🔴 `path:line` — issue + suggested fix.
- 🟡 `path:line` — issue + suggested fix.
- 🟢 `path:line` — nit.
</output>
```

The four checks to weigh: **correctness** (bugs, edge cases), **security** (injection, auth, secrets), **performance** (N+1, allocations, sync-on-hot-path), **maintainability** (naming, complexity, missing tests).

## Template — bug debugging

The three things every debugging prompt needs:

```text
<role>
You are an experienced {{language}} engineer debugging a production issue.
</role>

<inputs>
<expected>{{what the code should do}}</expected>
<actual>{{what the code does}}</actual>
<error>{{stack trace, error message, or "no error — silently wrong"}}</error>
<code language="{{lang}}">
{{minimal repro}}
</code>
<environment>{{language version, OS, key deps}}</environment>
</inputs>

<task>
1. Identify the most likely root cause.
2. Explain WHY it produces the observed symptom.
3. Propose ONE minimal fix.
</task>

<constraints>
- Cite a specific line of <code> as the cause.
- Do NOT propose architectural changes; this is a bug fix.
- If the cause cannot be determined from the inputs, state what's missing.
</constraints>

<output>
<root_cause line="{{line}}">...</root_cause>
<explanation>...</explanation>
<fix language="{{lang}}">...</fix>
<verification_step>How to confirm the fix works.</verification_step>
</output>
```

If the model can't find the cause, it's almost always because **expected**, **actual**, or **error** wasn't specified concretely. Tighten the inputs before re-prompting.

## Template — code generation

```text
<role>
You are a {{language}} engineer writing production code for {{project_type}}.
You follow the conventions linked below.
</role>

<task>{{one-sentence imperative}}</task>

<inputs>
<conventions>{{link or paste, e.g. naming, error handling, logging}}</conventions>
<api_contract>
Inputs:  {{types}}
Outputs: {{types}}
Errors:  {{error type and shape}}
</api_contract>
<existing_code language="{{lang}}">
{{relevant surrounding code, imports, helpers}}
</existing_code>
</inputs>

<constraints>
- Match the style and naming of <existing_code>.
- Handle these edge cases: {{list}}.
- {{tests required? docstrings? type hints?}}
- NEVER add dependencies not already in {{lockfile}}.
</constraints>

<output>
1. The new code, complete and runnable.
2. A brief note on any assumptions.
3. Suggested test cases (file + names + scenarios — not the bodies unless asked).
</output>
```

## Template — architecture proposal

```text
<role>
You are a software architect designing for {{scale + constraints}}.
</role>

<task>Propose an architecture for {{problem}}.</task>

<inputs>
<requirements>...</requirements>
<constraints>{{scale, latency, cost, team_size, deadline}}</constraints>
<existing_systems>{{what already exists}}</existing_systems>
<non_requirements>{{things this does NOT need to do — equally important}}</non_requirements>
</inputs>

<output>
## High-level diagram
(ASCII)

## Components
- Name: responsibility, technology, why this choice.

## Data flow
Step-by-step happy path.

## Failure modes
What breaks under {{network partition | dependency outage | high load}}.

## Trade-offs
Three alternatives considered and why rejected.

## Open questions
Things the human must decide.
</output>
```

## Template — test generation (AAA)

```text
<role>
You are a {{language}} engineer writing thorough, fast tests using {{framework}}.
</role>

<task>Write tests for the function below using the Arrange-Act-Assert pattern.</task>

<inputs>
<function language="{{lang}}">
{{function under test}}
</function>
<contract>
Inputs:  {{types and ranges}}
Outputs: {{types}}
Errors:  {{when}}
</contract>
</inputs>

<constraints>
- Cover: happy path, edge cases (empty, max, boundary), error cases.
- One behavior per test; descriptive test names.
- No mocks unless the function depends on external I/O.
- Use {{framework}}'s assertion style.
</constraints>

<output>
For each test:
1. Test name (describes the scenario in plain English).
2. Arrange: setup.
3. Act: call.
4. Assert: expectation.
</output>
```

## Template — refactor proposal

```text
<role>
You are a senior engineer proposing a refactor that ships in one PR.
</role>

<task>
Identify the smallest refactor that {{specific goal: reduces duplication |
clarifies the public API | unblocks {feature} | etc.}}.
</task>

<inputs>
<files>
{{paste of files in scope}}
</files>
<non_goals>
- Don't touch {{out-of-scope areas}}.
- Don't add abstractions for hypothetical future requirements.
- Don't change external behavior.
</non_goals>
</inputs>

<output>
## Smallest viable refactor
What to change, in 3 bullets.

## Diff sketch
Before/after for the 1–2 most important spans.

## What we're NOT doing
And why — the things a future PR could pick up.

## Risks
Tests that must pass; behavior to preserve.
</output>
```

The non-goals are the most important slot. Without them you get a redesign, not a refactor.

## Cost & model routing

| Task | Tier | Notes |
|---|---|---|
| Lint-style nits / minor reviews | cheap | Cached few-shot of style rules. |
| PR review on a small diff | mid | Streaming on. |
| Bug debug with full repro | mid → frontier | Frontier when symptom is non-obvious. |
| Architecture proposal | frontier | Worth the spend; rarely run. |
| Test generation | mid | Cache the framework conventions block. |

## Anti-patterns

- **No `<expected>` / `<actual>` / `<error>` in debug prompts.** You'll get speculation.
- **Reviewing prose changes in a code-review prompt.** Use the writing template.
- **Asking for "best practices."** Vague. Ask for "what would block this from merging in this codebase."
- **Generated code without a contract.** Match imports, types, errors, naming — or you get plausible code that doesn't compile.
- **Refactor prompts without `<non_goals>`.** Scope creeps; the diff explodes.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Prompt Frame.
- `jimmy-skills@prompt-engineering-output-xml` — preferred for code review (mixed prose + structure).
- `jimmy-skills@prompt-engineering-output-structured` — for human-rendered review summaries.
- `jimmy-skills@prompt-engineering-eval` — fixtures with known-buggy code.
