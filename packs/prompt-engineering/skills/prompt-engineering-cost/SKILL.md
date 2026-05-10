---
name: prompt-engineering-cost
description: "Reduce LLM cost and latency without sacrificing quality. Covers model routing, prompt-prefix caching, response-cache by hash, token compression, max_tokens caps, batch APIs, streaming, and the quality-vs-cost decision matrix. Use when production spend or p95 latency is the constraint."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Specific cache TTLs, batch discounts, and rate limits vary by provider — verify against current docs."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# Prompt Cost & Latency Engineering

> **Disclaimer.** Cache mechanics, batch discounts, and per-token pricing change frequently. Provider documentation is authoritative. The patterns below are stable; the numbers aren't.

The two production constraints are **dollars per request** and **wall-clock latency**. Both are knobs, not constants. This skill lays out the levers in priority order — start at the top.

## Lever 1 — Model routing

The single highest-leverage cost optimization: stop using the frontier model for tasks that don't need it.

| Task | Tier | Why |
|---|---|---|
| Classification, intent detection | cheap | Small, well-trained models match frontier on classification. |
| Extraction with a tight schema | cheap | Schema does the work. |
| Summarization (short → short) | cheap to mid | Match volume, not quality. |
| Drafting prose, code generation | mid | Quality matters but not frontier-level. |
| Multi-step reasoning, planning | frontier | Where the gap is largest. |
| Ambiguous synthesis, novel problems | frontier | Same. |
| Adversarial / safety-critical | frontier (with eval) | Wrong answers expensive. |

**Pattern: cheap classifier → tiered handler.**

```text
intent = haiku(classify_prompt, user_input)        # $
if intent.confidence > 0.8:
    answer = sonnet(handler_for_intent, …)         # $$
else:
    answer = opus(careful_handler, …)              # $$$
```

Most production traffic is the easy 80%. Routing alone often cuts cost 60–80%.

## Lever 2 — Prefix caching

The system prompt + role + tool definitions + few-shot block rarely changes between calls. Anthropic, OpenAI, and Google all support some form of prompt caching — the cached prefix is charged at a fraction of the input rate (typically 10%) and skipped on the wire.

```text
[ cached prefix (system + tools + few-shot) ]   ← billed at 10%, low latency
[ per-call payload (retrieval + user)       ]   ← billed at 100%
```

Practical rules:

- Move every stable token to the cached prefix.
- Avoid sprinkling per-call values into the system prompt; keep them in the user message.
- Cache TTLs are short (~5 minutes on Anthropic ephemeral cache). For low-traffic flows, top up the cache with a synthetic warmup call.
- Cache hit-rate is a first-class metric. Track it.

## Lever 3 — Response cache (by request hash)

For idempotent, deterministic queries (definitions, classifications, lookups), cache the entire response keyed by a hash of the inputs.

```python
key = hash(model + prompt_template_id + canonicalized_inputs)
if response := cache.get(key):
    return response
response = call_model(...)
cache.put(key, response, ttl=24h)
```

Works best when:

- Inputs have low cardinality.
- Output is short.
- Staleness is tolerable for the TTL window.

Combine with prefix caching — they compound.

## Lever 4 — Token compression

Shorter prompts cost less and run faster. Compress without losing meaning:

- Drop pleasantries ("please", "kindly", "thank you").
- Replace prose with structure (a 5-bullet schema beats a paragraph).
- Use abbreviations the model already knows (`PR`, `CI`, `LLM`, `RBAC`).
- Reference content by position (`<doc id="A">`) instead of repeating it.
- Trim few-shot to the smallest set that preserves quality (the 5th example is rarely worth its tokens).

The classic compression demo:

> 67-token verbose prompt → 12-token compressed prompt = **82% fewer tokens**, equivalent output quality.

The cap is quality, not creativity. Measure with `prompt-engineering-eval` after each compression.

## Lever 5 — Output bounds

Set `max_tokens` based on **what your downstream actually needs**, not on model max.

- Classification → 50 tokens.
- Short summary → 300 tokens.
- Code review → 1500 tokens.
- Long-form article → 4000 tokens.

Two effects:

1. Caps cost when the model loops or rambles.
2. Reduces p95 latency: most providers stream until `max_tokens`, so a tight cap is a tight latency contract.

Pair with explicit length constraints in the prompt ("≤200 words"). Belt and suspenders.

## Lever 6 — Streaming

Stream when:

- A human reads the output as it arrives (chat, IDE, doc gen).
- Time-to-first-token matters more than total latency.

Don't stream when:

- Code parses the full response (incomplete JSON is unparseable).
- You need the whole answer for validation before exposing.
- The output is short enough that streaming overhead exceeds the wait.

For chains, **stream the last step only.** Earlier steps need their full output for validation.

## Lever 7 — Batch APIs

Most providers offer batch endpoints with 50% off-peak discounts and async delivery (24h SLA). Use for:

- Backfills.
- Periodic reports.
- Large-scale labeling / extraction.

Don't use for: anything user-facing.

## Lever 8 — Parallelization

For chains where steps are independent, parallel calls cut wall-clock latency to the slowest branch. Cost stays roughly equal (you pay for the same work either way), but your p95 drops.

Watch out for:

- Provider rate limits (bursty traffic gets 429s).
- Tool calls with side effects — don't parallelize writes.

## The quality / cost / latency triangle

| Constraint | Levers |
|---|---|
| **Cheaper** | Model routing, prefix cache, response cache, compression, batch API, smaller few-shot |
| **Faster** | Smallest effective model, prefix cache, streaming, max_tokens cap, parallel branches |
| **More accurate** | Frontier model, CoT or reasoning model, self-consistency (n=3–5), validation step, longer context |

You cannot maximize all three. Pick the binding constraint, then optimize the other two within budget.

A useful default for an interactive product:

- **First pass**: cheap model, cached prefix, tight max_tokens, streaming on. Targets <2s p50, <$0.001/req.
- **Hard cases**: route to frontier with reasoning. Targets <10s p50, <$0.05/req.
- **Offline**: batch API on whatever model. Targets cheapest per token.

## Measuring

Instrument every call:

```text
input_tokens, cached_input_tokens, output_tokens, model, latency_ms,
status, retry_count, prompt_template_id, route_branch
```

Compute and dashboard:

- Cost per request, p50/p95.
- Latency, p50/p95.
- Cache hit-rate.
- Retry rate per template.
- % of traffic per route branch.

Without this, every optimization is a guess.

## Anti-patterns

- **"Just use the best model everywhere."** 5–10× cost for marginal quality on most tasks.
- **Sprinkling per-call values into the system prompt.** Defeats prefix caching.
- **Long pleasantries in the prompt.** Free tokens for the provider; pure cost for you.
- **No `max_tokens`.** A looping model can produce 10k tokens of garbage at full price.
- **Streaming when code parses the response.** Partial JSON is worse than no JSON.
- **Optimizing without measurement.** "I think this is faster" is not data.
- **Aggressive caching of nondeterministic outputs.** Stale, surprising answers; users notice.
- **Shipping the cheap-model route without an eval gate.** Cost savings + quality regression = loss.

## Quick decision tree

```text
Is the task user-facing and interactive?
├─ Yes → start cheap-tier with cached prefix and streaming
│         Route to frontier ONLY when classifier flags hard case
└─ No  → batch API, mid tier, no streaming, aggressive max_tokens

Is the prompt's prefix stable across calls?
├─ Yes → prefix-cache it (mandatory above ~500 tokens of stable content)
└─ No  → restructure until it is, or accept the cost

Are the inputs low-cardinality and deterministic?
├─ Yes → response-cache by request hash
└─ No  → skip response cache

Is variance high enough that wrong answers are expensive?
├─ Yes → self-consistency (n=3) and verification step
└─ No  → single call with confidence field
```

## Cross-references

- `jimmy-skills@prompt-engineering-context` — caching the prefix depends on a stable prefix architecture.
- `jimmy-skills@prompt-engineering-chain` — model tiering is mostly a chain decision.
- `jimmy-skills@prompt-engineering-eval` — gate cost optimizations behind eval regressions.
- `jimmy-skills@prompt-engineering-reasoning` — when frontier-with-reasoning is worth the spend.
- `jimmy-skills@prompt-engineering-few-shot` — prime target for compression.
