---
name: prompt-engineering-pitfalls
description: "Pre-flight checklist of nine production prompt anti-patterns: vagueness, overloading, assumption trap, leading questions, blind trust, one-shot mentality, format neglect, context-window abuse, and hallucination ignorance. Run this checklist before shipping any prompt and when debugging unreliable outputs."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. The pitfalls show up on every provider and every model size."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Glob Grep
---

# Prompt Engineering — Pitfalls Checklist

Most "the AI doesn't work" reports trace to one of nine reproducible anti-patterns. Run this checklist before shipping a prompt and again whenever a working prompt starts misbehaving in production.

## How to use this skill

- **Before shipping** — walk the nine items, fix the violations, then run your eval set.
- **When debugging** — start at the symptom, jump to the matching pitfall, apply the fix.
- **In code review** — paste this checklist into the PR template for prompt changes.

## The Nine Pitfalls

### 1. Vagueness

**Symptom:** Output varies wildly call-to-call. Generic, average-y answers.

**Cause:** Critical context is in your head, not in the prompt. "Write something about marketing" assumes the model knows your audience, length, channel, and goal.

**Fix:** Specify *audience*, *length*, *format*, and *goal* explicitly. The Prompt Frame's `<context>`, `<constraints>`, and `<output>` slots exist for this.

### 2. Overloading

**Symptom:** Output addresses some requirements and misses others. "Funny" overrides "professional." "Beginner-friendly" overrides "advanced."

**Cause:** Two competing objectives in one prompt. The model averages.

**Fix:** One objective per prompt. If you genuinely need two outputs, run two prompts (or a prompt chain — `prompt-engineering-chain`) and merge the results in code.

### 3. The Assumption Trap

**Symptom:** "Update the function I showed you earlier" returns a hallucinated function. Cross-session references fail.

**Cause:** Models are stateless across sessions. Even within a session, references to "the thing we discussed" are fragile because the relevant context may have been compressed.

**Fix:** **Re-state the artifact** every time you reference it. Wrap inputs in tags (`<previous_function>...</previous_function>`). For long-running agents, push prior artifacts through `prompt-engineering-context` — RAG or summarization, not implicit memory.

### 4. Leading Questions

**Symptom:** The model agrees with whatever framing you used. "Why is Python the best?" returns a defense of Python, not an analysis.

**Cause:** The question embeds the conclusion. The model continues the pattern you started.

**Fix:** Frame neutrally. "Compare Python, Go, and Rust for *<this specific task>*; recommend one with reasons." Ask for trade-offs explicitly. For analysis, request perspective roles (`prompt-engineering-role`) on each side and synthesize.

### 5. Trust Everything

**Symptom:** Confidently wrong statistics, fabricated citations, plausible-but-broken code merged into production.

**Cause:** Fluency is not accuracy. The model writes confidently regardless of confidence.

**Fix:** Require **evidence fields** alongside answers (`confidence`, `sources`, `evidence`). Run code before believing it. Verify citations. For high-stakes paths, add an explicit verification step in a chain.

### 6. One-Shot Mentality

**Symptom:** "It didn't work the first time, so AI is bad at this."

**Cause:** Treating prompts as fire-and-forget instead of as a system to be iterated.

**Fix:** Adopt the **Write → Test → Analyze → Improve** loop from `prompt-engineering-refine`. Change one variable per iteration. Keep a small fixture set (5–10 inputs) and re-run after every change.

### 7. Format Neglect

**Symptom:** Downstream parser breaks on a Tuesday because the model started wrapping its JSON in code fences this week.

**Cause:** The output format was a request, not a contract.

**Fix:** Use `prompt-engineering-output-json` or `prompt-engineering-output-xml`. Define a schema. Validate after parse. **Prompt to prevent, parse to recover** — do both, not either.

### 8. Context-Window Abuse

**Symptom:** Latency spikes, costs balloon, and the model "forgets" things it should know. Quality degrades on long inputs.

**Cause:** Pasting whole documents when the model needs only the relevant pages. Context windows are not "free RAM."

**Fix:** Chunk and retrieve relevant passages (RAG). Summarize earlier turns once they're load-bearing only as context. Place the most important content **at the start and end** of the window (models attend most strongly to the edges). See `prompt-engineering-context`.

### 9. Hallucination Ignorance

**Symptom:** A user catches the model citing a paper that doesn't exist, an API endpoint that was never built, a person who never said the quoted thing.

**Cause:** The model produces plausible continuations. Plausibility is its training objective; truthfulness is your problem.

**Fix:** For factual outputs, require **sources from a retrieved context** (RAG), not from the model's prior. For code, require runnable verification. State explicit "if unknown, return null" rules (see `prompt-engineering-output-json` null-over-invent). Track hallucination rate as a first-class eval metric.

## Symptom → Pitfall Map

| Symptom | Most likely pitfall |
|---|---|
| Output varies on identical input | #1 Vagueness, #7 Format Neglect |
| Some requirements ignored | #2 Overloading |
| References to past content fail | #3 Assumption Trap, #8 Context Abuse |
| Model agrees with bad framing | #4 Leading Questions |
| Confidently wrong | #5 Trust Everything, #9 Hallucination |
| "AI doesn't work for this" | #6 One-Shot Mentality |
| Parser breaks intermittently | #7 Format Neglect |
| Latency / cost growing over time | #8 Context Abuse |
| Fabricated facts / citations | #9 Hallucination |

## Pre-Ship Checklist

Run through this before shipping any prompt to production:

- [ ] Audience, length, format, and goal are explicit (#1).
- [ ] Exactly **one objective**; competing requirements split into separate prompts or chain steps (#2).
- [ ] Every artifact the prompt references is **re-stated** inside the prompt (#3).
- [ ] No leading framing; analysis prompts request trade-offs explicitly (#4).
- [ ] Output includes **confidence/sources/evidence** when factual claims matter (#5, #9).
- [ ] At least **5 fixtures** exist; the prompt has been iterated against them (#6).
- [ ] Output format is a **schema**, not a request, with a parser and validator (#7).
- [ ] Context includes only what the task needs; long inputs are chunked or summarized (#8).
- [ ] Factual outputs are grounded in retrieved context, not model prior (#9).
- [ ] Schema, parser, positive example, negative example, and known failure modes are committed alongside the prompt.

## When you're stuck

If the prompt is producing bad output and none of the pitfalls obviously apply:

1. **Print the full final prompt.** Variables that didn't interpolate, missing system messages, doubled instructions — half of "bad prompt" is actually "wrong prompt sent."
2. **Run it on a frontier model.** If it works there but fails on the cheap model, the issue is model capacity, not prompt design — see `prompt-engineering-cost` for the routing fix.
3. **Run it ten times.** If results vary, you have unspecified variance (pitfalls #1 / #7). If results are consistently wrong, you have a logic or context problem (#3 / #8 / #9).
4. **Have the model critique its own output.** Pass output back with "Identify the three biggest weaknesses and propose a fix to each." This often surfaces what's missing in the original instructions.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — the Prompt Frame and Output Contract.
- `jimmy-skills@prompt-engineering-system-prompt` — for persistent identity issues.
- `jimmy-skills@prompt-engineering-output-json` — for #7 fixes.
- `jimmy-skills@prompt-engineering-refine` — for #6 (iteration loop).
- `jimmy-skills@prompt-engineering-edge-cases` — empty / long / adversarial input.
- `jimmy-skills@prompt-engineering-context` — for #3 / #8 / #9.
