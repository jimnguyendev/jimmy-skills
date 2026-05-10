---
name: prompt-engineering-few-shot
description: "Show the model examples instead of describing them. Covers the 2–5 rule, diversity-over-quantity, negative examples, edge-case demonstrations, ordering effects, and the cost-vs-quality trade-off. Use when describing the desired output is harder than showing it, or when format and tone need to be tightly controlled."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Long-context models tolerate many-shot (10–100s) but most production prompts plateau at 3–5 examples."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Few-Shot Prompting

Showing examples is often a tighter contract than describing them. Examples encode four things at once that prose struggles to convey:

1. **Output structure** — exact field order, delimiters, indentation.
2. **Tone and vocabulary** — formal vs. casual, jargon density, hedging style.
3. **Reasoning pattern** — the path from input to output, not just the destination.
4. **Edge-case handling** — how to behave when the input is unusual.

If you find yourself writing increasingly detailed prose to specify formatting or style, stop and write three examples instead.

## When to use

- Output style is hard to describe but easy to recognize (brand voice, code-comment style, classification labels).
- Format must be exact (CSV with specific headers, a template the rest of your pipeline expects).
- You need consistent handling of edge cases the prose can't enumerate.
- Zero-shot results vary too much across calls.

**Don't use few-shot when:** the task is simple and zero-shot already works (you'd just be paying for tokens), the cost budget is tight and a description is sufficient, or you can use a provider's structured-output mode (`prompt-engineering-output-json`) to enforce shape.

## The 2–5 Rule

| Examples | Use when |
|---|---|
| 0 (zero-shot) | Simple tasks; the role + instructions already pin behavior. |
| 1 (one-shot) | Format is the only thing that matters; one canonical example is enough. |
| **2–5 (the sweet spot)** | Most production tasks: classification, extraction, transformation, generation. |
| 5+ (many-shot) | Long-context models, ambiguous categories, niche domain style. Diminishing returns past ~10 unless the task is genuinely high-variance. |

Practical defaults:

- Simple classification → 2–3 examples (one per category + one ambiguous).
- Complex formatting → 3–5 examples covering structure variants.
- Nuanced style (brand voice, persona) → 4–6 examples.
- Edge cases → add 1–2 alongside normal scenarios.

## Diversity beats quantity

Three diverse examples teach more than ten near-duplicates. Cover:

- Easy and hard inputs.
- Common and rare categories.
- Short and long inputs.
- Inputs where the answer is "none" or "unknown" (if applicable).

If your examples are all variations of the same input, you've built a regex, not a prompt.

## Negative examples

Show the model what *not* to do — explicitly labeled.

```text
<example>
Input: "I love this product, but the shipping took forever."
Output (correct): {"sentiment": "mixed", "topics": ["product", "shipping"]}
Output (wrong): {"sentiment": "positive"}
Reason wrong: Misses the negative shipping signal.
</example>
```

Negative examples are most useful when the model has a known failure mode (e.g. defaulting to "positive" because positive is the majority class in pretraining). They burn tokens, so reserve them for failure modes your eval set actually shows.

## Edge-case demonstrations

A few-shot prompt with only the happy path teaches the model the happy path is universal. Include 1–2 edge cases:

- Empty input → expected handler.
- Ambiguous input → request for clarification.
- Out-of-scope input → exact deflection.
- Multilingual / mixed input → expected normalization.

This is how you make few-shot prompts production-robust without the model inventing behavior on unfamiliar input.

## Format

Use a consistent example format. Either tagged blocks (Claude-friendly) or labeled markdown:

```text
<example>
<input>...</input>
<output>...</output>
</example>

<example>
<input>...</input>
<output>...</output>
</example>

Now do this one:
<input>{{user_input}}</input>
<output>
```

Or:

```text
Example 1:
Input: ...
Output: ...

Example 2:
Input: ...
Output: ...

Now:
Input: {{user_input}}
Output:
```

Pick one and use it consistently within a prompt. Mixing formats confuses the model about which delimiters mean "field boundary."

## Ordering effects

- **Recency bias.** Models attend more strongly to the most recent example. Place the most representative example **last**.
- **Pattern-match bias.** If your examples all happen to share a feature ("all 5-word inputs"), the model may treat that feature as a constraint. Vary length, structure, and content.
- **Easy → hard.** When using CoT-style few-shot, ordering from simple to complex helps the model build up the reasoning pattern.

## Cost & caching

- **Cache the few-shot block.** It's stable across calls; put it in the prefix-cached section of the system prompt.
- **Trim ruthlessly.** Every example is paid for on every call. If a 5th example doesn't measurably improve eval scores, drop it.
- **Reuse across prompts.** A shared few-shot library used by multiple prompts amortizes the curation cost.

## Anti-patterns

- **Inconsistent formatting across examples.** If one example uses `Output:` and another uses `Answer:`, the model learns the inconsistency.
- **Examples that disagree.** Two examples with similar inputs but different outputs = noise. Resolve the conflict in your training data first.
- **Examples leaking real PII or secrets.** They go into prompt logs and prompt caches. Anonymize.
- **Stale examples.** When categories or schemas evolve, examples drift. Treat the few-shot block as code; review on each schema change.
- **Too many examples on a small model.** Long context degrades quality on smaller models; 3 sharp examples beat 8 diluted ones.

## Quick template

```text
<role>...</role>
<task>Classify the input into one of: A, B, C, or "unsure".</task>

<examples>
<example><input>...</input><output>A</output></example>
<example><input>...</input><output>B</output></example>
<example><input>...</input><output>"unsure"</output></example>   <!-- edge case -->
<example><input>...</input><output>C</output></example>           <!-- representative, last -->
</examples>

<input>{{user_input}}</input>
<output>
```

## Cross-references

- `jimmy-skills@prompt-engineering-core` — the Prompt Frame.
- `jimmy-skills@prompt-engineering-role` — combine role + few-shot for tone-heavy tasks.
- `jimmy-skills@prompt-engineering-reasoning` — CoT few-shot for multi-step problems.
- `jimmy-skills@prompt-engineering-refine` — single-variable iteration when adding/removing examples.
- `jimmy-skills@prompt-engineering-cost` — caching strategy for the few-shot block.
