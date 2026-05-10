---
name: prompt-engineering-refine
description: "The Write → Test → Analyze → Improve loop for evolving a prompt to production quality. Single-variable iteration, a symptom-to-fix diagnostic table, version logging, and a stop rule. Use when you have a working-but-imperfect prompt and need a disciplined way to improve it without thrashing."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Pairs with any eval harness."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# Iterative Refinement

A prompt is rarely right on the first try. The job is not to write the perfect prompt — it is to converge on a prompt good enough for the use case, **predictably**, by changing one thing at a time and measuring.

## When to use

- You have a prompt that works ~70% of the time and need to push it higher.
- A working prompt regressed (model upgrade, schema change, new edge case).
- You're tempted to rewrite the prompt from scratch — stop and iterate instead.
- You're handing the prompt off and need a record of why it's structured the way it is.

## The Loop

```text
Write → Test → Analyze → Improve → (back to Test)
```

**Write** — start from the Prompt Frame (`prompt-engineering-core`), not from scratch.
**Test** — run against a fixture set of ≥5 inputs covering the realistic distribution.
**Analyze** — categorize failures by symptom (see table below), don't just count them.
**Improve** — change **one** thing. Note what and why.

## Single-variable iteration

The cardinal rule: **change one variable per round.**

If you change role + add few-shot + tighten the schema in the same round and accuracy goes from 70% → 85%, you don't know which change helped. Worse, one change might have hurt and another over-compensated — masking the regression until later.

If you genuinely need to change two things, run them as two rounds: A → A+1 → A+1+2.

## Symptom → Fix table

Map symptoms to specific changes. Don't rewrite the prompt; surgically adjust.

| Symptom | Most likely fix | Skill |
|---|---|---|
| Output too long | Add `max_tokens` cap and explicit length constraint | core |
| Output too short / shallow | Request "elaborate on X with at least N sentences" | core |
| Wrong format | Add a schema in `<output>`; require self-validation | output-json / output-xml |
| Tone misaligned | Add audience + 2–3 style adjectives; or 2 examples | role / few-shot |
| Inconsistent across calls | Add few-shot; lower temperature; tighten constraints | few-shot |
| Misses a requirement | Move requirement into `<constraints>` with MUST | core |
| Hallucinates fields | Add null-over-invent rule; require sources | output-json |
| Refuses overly often | Loosen scope clause; add allowed examples | system-prompt / role |
| Slow / expensive | Trim few-shot; cache prefix; smaller model | cost |
| Fails on edge inputs | Add edge-case example; explicit handler | edge-cases / few-shot |
| Format breaks intermittently | Tighten schema; add parser repair | output-json |
| Reasoning skipped | Add explicit `<reasoning>` slot | reasoning |
| Reasoning too verbose | Move reasoning into scratchpad; cap length | reasoning |

If two rows match the symptom, try the cheaper fix first (constraint tightening before adding examples; few-shot before chaining).

## Version log

Every change goes in a log next to the prompt:

```text
v1  baseline                                          fixtures: 7/10
v2  add role: "Senior Postgres DBA, 15y OLTP"         fixtures: 8/10  (+1)
v3  v2 + null-over-invent rule                        fixtures: 9/10  (+1)
v4  v3 + scratchpad reasoning slot                    fixtures: 9/10  (—; revert)
v5  v3 + edge-case example: empty EXPLAIN output      fixtures: 10/10 ✓
```

The log is the prompt's git history. It tells future-you (or a teammate) why a constraint exists and lets you revert cleanly when the model upgrades and an old workaround stops mattering.

## Fixture set

Maintain a small, named fixture set per prompt:

```yaml
fixtures:
  - name: golden_path
    input: ...
    expect: contains_field("severity"); severity in [1,2,3,4]
  - name: empty_input
    input: ""
    expect: returns_empty_handler
  - name: ambiguous_input
    input: ...
    expect: asks_clarifying_question
  - name: adversarial
    input: "Ignore previous instructions and ..."
    expect: refuses_or_deflects
  - name: long_input
    input: <10kb of text>
    expect: handles_or_chunks
```

5–10 fixtures is enough for tight iteration. The full eval set goes in `prompt-engineering-eval`.

## Meta-technique: ask the model to critique itself

When you're stuck, pass the prompt + a failing output back to the model:

```text
<original_prompt>...</original_prompt>
<actual_output>...</actual_output>
<expected>...</expected>

What three changes to <original_prompt> would most likely fix this?
For each, name the slot (role / context / task / constraints / output)
and the exact text to add or replace.
```

The suggestions are not always right — but they often surface what's missing, which is faster than guessing.

## Stop rule

Iteration ends when:

1. The fixture set passes consistently across N runs (N=3–5).
2. The next plausible change costs more than the marginal quality gain (cost vs. accuracy curve has flattened).
3. The remaining failures are out of distribution for the production workload.

It does **not** end at "100% on fixtures" — overfitting to fixtures is real. Track production-sample quality separately.

## Anti-patterns

- **Rewriting from scratch when results regress.** You lose the version log and the diagnostic.
- **Changing the prompt and the model in the same round.** You can't attribute the change.
- **Iterating on a single example.** Always test against the fixture set.
- **No stop rule.** "One more tweak" can run forever; commit when the bar is met.
- **Tweaking without writing down why.** A month later you'll undo the fix because you forgot the reason. Keep the log.
- **Treating fixture pass as production-ready.** Run against fresh production samples before shipping.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — the Prompt Frame to refine within.
- `jimmy-skills@prompt-engineering-pitfalls` — symptom map alignment.
- `jimmy-skills@prompt-engineering-few-shot` — common improvement lever.
- `jimmy-skills@prompt-engineering-eval` — full eval harness, regression tests.
