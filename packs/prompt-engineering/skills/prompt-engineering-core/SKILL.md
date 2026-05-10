---
name: prompt-engineering-core
description: "Foundation of the prompt-engineering pack. Defines the Prompt Frame (7 canonical slots), the Output Contract, the Cost Profile, and the routing map to every other skill in the pack. Use FIRST when starting any prompt design work or when unsure which prompt-engineering skill to reach for."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Works with Claude, GPT, Gemini, and open-weight models. Examples lean on Claude conventions where they materially differ."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep AskUserQuestion
---

# Prompt Engineering — Core

> **Disclaimer.** Patterns here distill widely-published prompt-engineering guidance (Anthropic, OpenAI, Google) and practitioner experience. Provider documentation remains authoritative for model-specific behavior, context limits, and pricing.

The foundation skill. Three reusable abstractions and a routing map. Every other skill in this pack composes on top of these.

## Core Principle

A prompt is a **contract**, not a wish. It specifies who is speaking (role), what they know (context), what they must produce (task), in what shape (output), and within what limits (constraints). Anything not specified is left to the model's prior — which means it varies between calls, between models, and across versions.

The job of prompt engineering is to **remove unspecified variance** until the output is reliable enough for the use case, and no further.

## The Prompt Frame (7 slots)

Every prompt in this pack is built by filling the same seven slots, in this order. Slots that don't apply are omitted, not left blank.

```text
<role>         Who the model is acting as.            → prompt-engineering-role
<context>      Background the task depends on.        → prompt-engineering-context
<task>         One imperative sentence.                One objective per prompt.
<inputs>       Variables, clearly delimited.           Use <tags> or ```fences```.
<constraints>  MUST / NEVER / length / scope rules.    Hard rules, not preferences.
<reasoning>    Optional. Triggers CoT.                 → prompt-engineering-reasoning
<output>       Schema or format contract.              → prompt-engineering-output-* skills
```

**Why this order matters.** Models attend most strongly to the start (role/context) and the end (output format). The task sits in the middle to keep it framed. Constraints come before reasoning so the model "thinks" within them, not around them.

**Compose by slot, not by rewriting.** When a domain skill (e.g. `domain-coding`) ships a template, it fills the slots — it never re-invents the structure.

### Minimal example

```text
<role>Senior Postgres DBA, 15 years on OLTP systems.</role>

<task>Diagnose the cause of the slow query below and propose one fix.</task>

<inputs>
<query>SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 50;</query>
<explain_analyze>...paste here...</explain_analyze>
</inputs>

<constraints>
- MUST cite a specific line of the EXPLAIN output as evidence.
- MUST propose exactly ONE fix (most-likely cause first).
- NEVER suggest hardware changes.
</constraints>

<output>
Return XML:
<diagnosis>...</diagnosis>
<evidence>...</evidence>
<fix sql="..."/>
</output>
```

That same skeleton works for code review, customer email triage, lesson generation, image-prompt design — change the slots, not the structure.

## The Output Contract

Any prompt that produces machine-consumed output must publish four things together:

1. **Schema** — JSON Schema, Pydantic/Zod model, XSD, or a typed XML example.
2. **Parser** — the exact code (5–15 lines) that turns the model's text into a typed value.
3. **One positive + one negative example** — what valid looks like, and what a common failure looks like.
4. **Failure-mode notes** — known ways this contract breaks (markdown wrapping, trailing commas, unescaped quotes, hallucinated enum values).

If you cannot write all four, the prompt is not yet production-ready. Skills `prompt-engineering-output-json` and `prompt-engineering-output-xml` ship the templates.

## The Cost Profile

Every reusable template is labeled with:

```yaml
model_tier:        cheap | mid | frontier
est_input_tokens:  <int>          # at the median call
est_output_tokens: <int>          # bounded by max_tokens
cache_strategy:    none | prefix | full
latency_budget_ms: <int>
```

Routers and `prompt-engineering-cost` read this to pick a model, set `max_tokens`, and decide whether to cache the stable prefix (system + role + few-shot) so only the user turn varies on each call.

A useful default split:

- **Classification, extraction, formatting** → `cheap`, prefix-cached, low `max_tokens`.
- **Drafting, summarization, code generation** → `mid`, prefix-cached, streaming.
- **Multi-step reasoning, architecture, ambiguous synthesis** → `frontier`, prefix-cached, optional self-consistency.

## Routing Map

When you reach for prompt engineering, route by intent:

| Intent | Skill |
|---|---|
| "How do I structure this prompt?" | `prompt-engineering-core` (you are here) |
| Persistent agent identity | `prompt-engineering-system-prompt` |
| Focus a single message on an expert lens | `prompt-engineering-role` |
| Output must be parseable JSON | `prompt-engineering-output-json` |
| Long context / nested documents on Claude | `prompt-engineering-output-xml` |
| Output-format avoidance checklist | `prompt-engineering-pitfalls` |
| Show examples instead of describing them | `prompt-engineering-few-shot` |
| Hard reasoning / multi-step math / planning | `prompt-engineering-reasoning` |
| Iterating an existing prompt | `prompt-engineering-refine` |
| Empty / ambiguous / adversarial inputs | `prompt-engineering-edge-cases` |
| Multi-step pipeline | `prompt-engineering-chain` |
| RAG, summarization, MCP | `prompt-engineering-context` |
| Cost or latency is the constraint | `prompt-engineering-cost` |
| Regression tests for a prompt | `prompt-engineering-eval` |
| Tool-using agent loop | `prompt-engineering-agent` |
| Image / audio / video input | `prompt-engineering-multimodal` |

## Defaults to enforce

Adopt these unless a specific skill says otherwise:

1. **One objective per prompt.** Two competing objectives become "funny yet professional" and the model picks the average.
2. **Schema before prose.** Define the output contract first; write the prompt to satisfy it.
3. **Single-variable iteration.** Change one slot per revision so you can attribute the result.
4. **Cache the stable prefix.** System + role + few-shot rarely changes; put it where caching helps.
5. **Validate intermediate outputs in chains.** Don't propagate garbage.
6. **Express uncertainty explicitly.** A `confidence` or `sources` field beats a confidently-wrong sentence.
7. **System prompt under 500 words.** Push detail into references and load via RAG.

## When NOT to add prompt engineering

- The task is a one-off and the result will be human-read and human-edited. A plain instruction is fine.
- The model already produces correct output across your fixtures. Adding role/CoT/few-shot will only spend tokens.
- The variance you see comes from missing context, not unclear instructions. Reach for `prompt-engineering-context` instead.

## Cross-references

- `jimmy-skills@prompt-engineering-system-prompt` — for persistent identity.
- `jimmy-skills@prompt-engineering-role` — for one-shot expert framing.
- `jimmy-skills@prompt-engineering-output-json` — for parseable outputs.
- `jimmy-skills@prompt-engineering-output-xml` — for Claude-native tagged outputs.
- `jimmy-skills@prompt-engineering-pitfalls` — pre-flight checklist.
