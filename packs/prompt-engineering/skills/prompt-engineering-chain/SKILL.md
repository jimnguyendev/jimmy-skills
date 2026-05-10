---
name: prompt-engineering-chain
description: "Decompose hard tasks into specialized prompt steps. Covers sequential, parallel, conditional, and iterative chain shapes; intermediate validation; retry budgets; and the Extract → Transform → Generate baseline. Use when one prompt is doing too much, when reliability is the bottleneck, or when different parts of the task want different models."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Pairs with provider-side parallel tool calling, batch APIs, and orchestration libraries (LangGraph, Mastra, in-house DAGs)."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# Prompt Chaining

> **Disclaimer.** Patterns are independent of orchestration framework (LangGraph, Mastra, custom DAGs) — those libraries' docs are authoritative for their APIs.

A single prompt forces one model to understand, plan, and generate at once. A chain breaks the work into specialized steps, each focused on one job. Each step is cheaper to debug, easier to swap models on, and produces an intermediate artifact you can validate before moving on.

## When to chain

- One prompt is producing inconsistent results because it has competing objectives.
- Different parts of the task want different model tiers (cheap classifier → frontier writer).
- You need to validate intermediate output (a parsed JSON, a fact-checked claim) before committing.
- The pipeline must branch (route by intent, by language, by severity).
- The task is naturally iterative: draft → critique → revise.

Don't chain when one prompt already works at acceptable quality and cost — chains multiply latency.

## Four chain shapes

### Sequential

Each step depends on the previous. The default. The classic baseline:

```text
Step 1: Extract  — pull the relevant facts from raw input.
Step 2: Transform — normalize / classify / enrich the facts.
Step 3: Generate — produce the user-facing output.
```

Why it works: each step has one job. The Extract step doesn't try to be eloquent. The Generate step doesn't try to parse.

### Parallel

Same input, multiple prompts running concurrently, results merged:

```text
input → [reviewer_security, reviewer_performance, reviewer_correctness] → merge
```

Use when:

- The lenses are independent (security review doesn't need to read the perf review).
- You want different roles / models per branch (security on frontier, style on cheap).
- Latency budget is tight — parallel is wall-clock fast.

The merge step is itself a prompt: *"Below are three reviews. Combine into one ordered list of issues, deduplicating overlap, severity-ranked."*

### Conditional

Route by classification:

```text
classify(input) → "billing"    → billing_handler
                  "technical"  → technical_handler
                  "account"    → account_handler
                  else         → fallback_handler
```

The classifier is small and cheap. The handlers can be specialized. This is how you get the right behavior without a 2000-word system prompt covering every case.

### Iterative

Generate → critique → revise → loop until quality threshold or max-iterations:

```text
draft = generate(prompt)
for i in 1..N:
    critique = evaluate(draft)
    if critique.passes: break
    draft = revise(prompt, draft, critique)
```

Use for: long-form writing, code generation with a test harness, design proposals. Cap N to keep cost predictable. The critique prompt is a separate, often cheaper, model.

## The Extract → Transform → Generate baseline

When in doubt, start here. It works for most content tasks:

```text
EXTRACT  (cheap, deterministic)
  Input: raw text / document / log / transcript.
  Output: structured JSON of facts. Schema validated.

TRANSFORM (cheap, deterministic)
  Input: structured JSON.
  Output: enriched / normalized / classified JSON.
  Validation: schema + business rules.

GENERATE (mid or frontier, generative)
  Input: enriched JSON.
  Output: user-facing prose / markdown / email / report.
  Constraints: tone, length, citations.
```

Most "make it more reliable" requests are answered by splitting a single mega-prompt into this shape. The Extract and Transform steps fail loudly (schema errors); only Generate operates on clean inputs.

## Intermediate validation

The cardinal rule: **never let unvalidated model output become the input to the next step.**

```python
def step(prompt, input_value, schema):
    raw = call_model(prompt, input_value)
    parsed = schema.validate(raw)        # raises on failure
    return parsed
```

Three failure modes to guard:

1. **Schema mismatch** — caught by Pydantic / Zod validation.
2. **Semantic drift** — the JSON parsed but values are nonsense; add domain assertions (`assert severity in {1,2,3,4}`).
3. **Hallucinated content** — the field exists but the value isn't grounded; require a `source` or `evidence` field and verify against the original input.

If a step fails validation, prefer **repair** (pass the error back, retry once) over **escalate** (fall back to a stronger model) over **fail** (return a `partial` result with an explanation).

## Retry budgets

Every step has a max retry count. Default: 1 repair retry, then escalate or fail.

```text
budget = { extract: 1, transform: 1, generate: 2 }
total_max_calls_per_request = sum(budget) + len(steps) = 7
```

Without a budget, a transient model error becomes a retry storm. Log the budget consumption per request so you can spot regressions (a step that suddenly retries 100% of the time).

## Streaming inside chains

Stream the **last step only.** Earlier steps need their full output for validation; partial JSON is unparseable. The user sees the final stream, not the intermediate stalls.

For agent loops, stream tool-call thinking as a "thinking…" indicator and stream the final natural-language summary.

## Cost levers

A chain costs more than a single prompt by default. Get the cost back with:

- **Model tiering.** Cheap model for Extract/Classify; frontier for Generate. The 80/20 of cost savings.
- **Cache the stable prefix per step.** System+role+few-shot rarely changes; only the input does.
- **Skip steps when possible.** If Extract finds no relevant facts, short-circuit before Generate.
- **Parallel where independent.** Wall-clock latency stays flat as you add concurrent branches.
- **Batch APIs for non-interactive workloads.** 50% off-peak discounts on most providers.

## Anti-patterns

- **Chains-as-fashion.** A two-step chain where one step would do — pure overhead.
- **Passing raw model text downstream.** Always validate first; chains amplify garbage.
- **No timeout per step.** A stuck call on step 2 hangs the whole pipeline.
- **Step-level prompts duplicating the system prompt.** Cache the system prompt; inject only the per-step instruction.
- **Hidden coupling between steps.** Step 3 depends on a field Step 1 happens to emit but isn't in the schema. Make the contract explicit in the schema.
- **Iterative chains with no max-iterations.** Cost runaway. Cap N; degrade gracefully when cap hit.
- **Same model for all steps.** You're leaving money on the table — and frontier models can be worse at tight extraction than purpose-tuned small ones.

## Quick template (sequential)

```python
extracted   = step("extract.prompt",  raw_input,    ExtractSchema,   model="haiku")
enriched    = step("transform.prompt", extracted,   EnrichedSchema,  model="haiku")
final       = step("generate.prompt",  enriched,    GenerateSchema,  model="sonnet")
return final
```

Each `prompt` file is a Prompt-Frame template. Each `Schema` is the Output Contract. Each call has its own retry budget and cache key.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Prompt Frame and Output Contract.
- `jimmy-skills@prompt-engineering-output-json` — schema validation between steps.
- `jimmy-skills@prompt-engineering-cost` — tiering, caching, batching.
- `jimmy-skills@prompt-engineering-eval` — fixture tests per step + end-to-end.
- `jimmy-skills@prompt-engineering-agent` — when chains become loops with tools.
- `jimmy-skills@prompt-engineering-edge-cases` — failure modes between steps.
