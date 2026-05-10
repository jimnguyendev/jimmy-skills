---
name: prompt-engineering-reasoning
description: "Make the model's reasoning visible to improve accuracy on multi-step tasks. Covers zero-shot CoT, few-shot CoT, the BREAK and Given→Goal→Approach→Steps→Verify templates, scratchpad patterns, self-consistency, and when reasoning is overhead rather than value. Use for math, debugging, planning, and anything that requires verifiable steps."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Modern reasoning models (Claude with extended thinking, OpenAI o-series, DeepSeek-R1) handle reasoning natively — prefer the provider's native reasoning mode when available; the prompt patterns here still apply for non-reasoning models or for shaping the visible answer."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Reasoning Prompts (Chain of Thought)

> **Disclaimer.** Modern reasoning-tuned models (Claude extended thinking, OpenAI o-series, DeepSeek-R1, Gemini thinking) handle multi-step reasoning natively. When using one, prefer the provider's reasoning mode and reserve these patterns for shaping the *visible* output. The provider's documentation is authoritative.

Chain-of-thought prompting asks the model to make its reasoning explicit before producing the final answer. This trades tokens for accuracy on tasks where the right answer depends on intermediate steps the model would otherwise skip.

## When to use

CoT helps:

- **Multi-step math** and unit conversions.
- **Logic puzzles** and constraint satisfaction.
- **Code debugging** by tracing execution.
- **Planning** with dependencies.
- **Decision-making** with explicit trade-offs.
- **Extraction** from unstructured text where the answer requires linking multiple sentences.

CoT does not help (and wastes tokens):

- Simple factual recall.
- Direct translation.
- Summarization.
- Creative writing where reasoning isn't the point.
- Anything a smaller model already gets right.

## Three approaches

### Zero-shot CoT

Add a trigger phrase. Cheapest and easiest:

```text
Let's think step by step.
```

Variants that often work better than the literal cliché:

- "Work through this carefully, showing each step."
- "Before answering, list what you know, what you need, and the steps to bridge them."
- "Identify the sub-problems first, then solve each, then combine."

Use zero-shot CoT when you don't care about the format of the reasoning, only its presence.

### Few-shot CoT

Provide worked examples. Each example shows input → reasoning → answer. The model mimics the demonstrated pattern.

```text
<example>
<input>A train leaves at 9:00 going 60 mph. Another at 9:30 going 80 mph. When does the second catch up?</input>
<reasoning>
Head start = 30 min × 60 mph = 30 miles.
Closing speed = 80 - 60 = 20 mph.
Time to close = 30 / 20 = 1.5 hours after 9:30.
</reasoning>
<answer>11:00</answer>
</example>
```

Use few-shot CoT when the *style* of reasoning matters: which steps to show, which to skip, what level of detail.

### Structured CoT (templates)

Force the reasoning into a fixed shape so it's parseable and consistent.

**BREAK:**

- **B**egin — restate the problem in your own words.
- **R**eason — list the key facts and constraints.
- **E**xecute — work through the solution.
- **A**nswer — state the final result.
- **K**now — note assumptions and uncertainties.

**Given → Goal → Approach → Steps → Verify:**

```text
<reasoning>
Given: <inputs and constraints>
Goal:  <what we are solving for>
Approach: <high-level strategy>
Steps:
1. ...
2. ...
3. ...
Verify: <sanity check, edge cases, alternative answer>
</reasoning>
<answer>...</answer>
```

Templates are best when the prompt runs in production and downstream tools want to extract the reasoning (e.g. for audit logs).

## Scratchpad pattern

Separate reasoning from the user-visible answer:

```text
<scratchpad>
Step through the problem here. This block will be discarded server-side.
</scratchpad>

<answer>
Final answer to surface to the user.
</answer>
```

Strip `<scratchpad>` before display. You get CoT accuracy without dumping the model's working onto the user. Pair with `prompt-engineering-output-xml` for clean parsing.

On reasoning-tuned models (extended thinking, o-series), the provider already handles this internally — don't double up.

## Self-consistency

For high-stakes outputs, generate **N independent solutions** (same prompt, non-zero temperature) and **vote / pick the majority** answer.

```text
N = 3 to 5
For each i in 1..N:
   draft_i = call_model(prompt, temperature=0.7)
final = majority(draft_1..draft_N)
```

Cost is N×; reserve for:

- Decisions you would re-run by hand to double-check.
- High-variance outputs (the same prompt produces different answers).
- Tasks where wrong answers are expensive (legal, financial, medical triage).

For multi-choice or extraction tasks, voting is straightforward. For generative tasks, use a follow-up prompt to pick the best of N: *"Below are N drafts. Pick the strongest and explain why in one sentence."*

## Cost & latency

- **Reasoning is paid for in output tokens.** Bound `max_tokens` and audit average reasoning length.
- **Reasoning models charge for hidden reasoning tokens.** Check your provider's billing.
- **Prefer scratchpad + truncate** over uncapped CoT in production: you control the answer length the user sees.
- **Stream the answer, not the scratchpad.** Hide the working until complete.
- **Self-consistency multiplies cost N×.** Use it sparingly and only where eval data shows it actually changes outcomes.

## Anti-patterns

- **CoT on tasks that don't benefit.** Adds latency and tokens for no quality gain.
- **Free-form reasoning when you need parseable steps.** Use the structured templates.
- **Visible reasoning in user-facing UIs without consent.** Users either get confused or learn to ignore it; consider scratchpad + clean answer.
- **Self-consistency without voting logic.** N drafts with no aggregation is just N× cost.
- **Mixing native reasoning + manual CoT.** On reasoning-tuned models the prompt-level "think step by step" is redundant and may interfere with the provider's tuning.
- **Trusting the reasoning as evidence.** A confident chain of thought can still arrive at a wrong answer; verification is separate.

## Quick template

```text
<role>...</role>
<task>...</task>
<inputs>...</inputs>
<constraints>...</constraints>

<reasoning>
Given: ...
Goal: ...
Approach: ...
Steps:
1. ...
Verify: ...
</reasoning>

<answer>
...
</answer>
```

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Prompt Frame; reasoning is the optional 6th slot.
- `jimmy-skills@prompt-engineering-few-shot` — pair with CoT for style-controlled reasoning.
- `jimmy-skills@prompt-engineering-output-xml` — scratchpad/answer separation.
- `jimmy-skills@prompt-engineering-cost` — when to use cheap-non-reasoning vs. frontier-reasoning.
- `jimmy-skills@prompt-engineering-eval` — measure whether CoT actually helps your task.
