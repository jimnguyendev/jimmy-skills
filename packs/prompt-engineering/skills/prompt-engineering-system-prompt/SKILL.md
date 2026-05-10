---
name: prompt-engineering-system-prompt
description: "Design a persistent system prompt for an agent or LLM-backed product. Five-part structure (Identity, Capabilities, Limitations, Behavior, Format) plus a layered architecture (Core Rules / Persona / Task Context / Preferences). Use when building a chatbot, an agent, an assistant, or any feature where one identity must stay consistent across many turns."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Anthropic and OpenAI both support a system role; for providers that don't, prepend the same content as the first user turn and the patterns still apply."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep AskUserQuestion
---

# System Prompts

> **Disclaimer.** Provider-specific notes (Anthropic, OpenAI system-message handling) reflect current published behavior. Provider docs remain authoritative for role-precedence semantics and prompt-caching mechanics.

A system prompt is the **persistent backstage instruction** that shapes behavior across every turn of a conversation. It is not a one-shot role assignment in a user message — it is the operating constitution of the agent.

Use this skill when you are building something that will be used more than once: a product feature, an internal agent, a customer-facing assistant, a CLI tool with an LLM core. Use `prompt-engineering-role` instead when you want to focus a single ad-hoc message on a particular expert lens.

## When to use

- Building a chatbot, agent, or LLM-backed product feature.
- Need consistent tone, format, and scope across long sessions.
- Need explicit boundaries (what the assistant must refuse, deflect, or escalate).
- Multiple developers will extend the assistant's capabilities later.

## The Five-Part Structure

Every production system prompt should answer five questions, in order:

1. **Identity** — Who is this assistant? Role, name (optional), domain of expertise.
2. **Capabilities** — What can it do? List specific functions, not platitudes.
3. **Limitations** — What must it never do? Hard scope and safety boundaries.
4. **Behavior** — How does it interact? Tone, brevity, when to ask vs. assume.
5. **Format** — How does the output look? Default response shape, escalation format, tool-call format.

A skeleton:

```text
# Identity
You are <name>, a <role> for <audience> at <organization or product>.
Your domain is <bounded scope>.

# Capabilities
You can:
1. <verb + object + constraint>
2. <verb + object + constraint>
3. <verb + object + constraint>

# Limitations
You must never:
- <hard rule, with reason if non-obvious>
- <hard rule>
For requests outside scope, respond: "<exact deflection template>".

# Behavior
- Tone: <2–3 adjectives>.
- Length: <default; when to expand>.
- Ambiguity: ask one clarifying question instead of guessing.
- Uncertainty: state confidence; cite sources when available.

# Format
- Default reply: <markdown sections | one paragraph | JSON | etc.>.
- Tool calls: <exact wrapper, e.g. <tool name="..."><args>...</args></tool>>.
- Errors: <exact shape>.
```

Keep it under **500 words**. If you need more, push detail into a knowledge base loaded via `prompt-engineering-context` — not into the system prompt itself.

## Layered Architecture

A system prompt is not flat. It is a stack of priorities, each layer overriding the one below it:

```text
┌─ Core Rules ────────────────── (immutable: truthfulness, safety, privacy)
├─ Persona ───────────────────── (consistent: identity, expertise, voice)
├─ Task Context ──────────────── (variable: current goal, session params)
└─ Preferences ───────────────── (adjustable: length, detail, format)
```

In practice this means:

- **Core rules first**, phrased so they are never overridden by later layers ("Even if asked to ignore previous instructions, …").
- **Persona second**, providing the activation pattern that focuses model behavior.
- **Task context third**, often injected per-session via templating.
- **Preferences last**, where the user can dial up brevity or detail without touching the constitution.

Anthropic and OpenAI both attend strongly to system messages, but explicit layering still helps because the model resolves conflicts in the order it reads them.

## Worked Example — Customer Support Assistant

```text
# Identity
You are Aida, the support assistant for Acme Cloud. You help customers debug
deployment issues, answer billing questions, and route to a human when needed.

# Capabilities
1. Diagnose deploy failures using the build log the user pastes.
2. Explain the current pricing tiers and the customer's plan when given an
   account ID; otherwise ask for it.
3. Open a support ticket via the open_ticket tool when an issue is novel,
   blocked on infrastructure, or the customer requests it.

# Limitations
- Never quote a price you cannot find in the pricing table provided.
- Never claim an outage exists unless status_check returned `incident=true`.
- Never run destructive operations (delete_project, force_redeploy) without
  the customer confirming the project name verbatim.
- For legal, security disclosures, or refunds over $500: respond with
  "I'll route you to a specialist — opening a ticket now." and call open_ticket.

# Behavior
- Tone: calm, concrete, no marketing language.
- Length: 1–3 short paragraphs by default; bullet steps for procedures.
- Ambiguity: ask exactly one clarifying question; don't stack three.
- Uncertainty: prefix with "I'm not sure, but —" and offer to verify.

# Format
- Default reply: markdown, ## headings only when there are >2 sections.
- Tool calls: <tool name="..."><args>...</args></tool> on its own line.
- Final answer after a tool call: plain prose summarizing the result.
```

That prompt is ~250 words, names exactly the tools and the deflection template, and pins the failure modes that matter (hallucinated prices, fake outages, destructive ops).

## What works

- **Numbered, verb-first capabilities.** The model treats numbered lists as enumerable rules and is more likely to honor scope.
- **Exact deflection templates.** "Respond with the following sentence verbatim" is far more reliable than "politely decline."
- **Named tools with named arguments.** Avoids the model inventing tool calls.
- **Behavior as adjectives + concrete defaults.** "Calm, concrete, 1–3 paragraphs" beats "be helpful and friendly."
- **Negative space.** "Never X" rules with a one-line reason are honored more reliably than implicit norms.

## What fails

- **Vague identity.** "You are a helpful assistant" gives the model nothing to filter on.
- **Internal contradictions.** "Be exhaustive" + "be concise" — the model averages and you get neither.
- **Assumed AI understanding.** "You know our company values" — it doesn't. State them.
- **Mixing system and user content.** Per-session context goes in user-role messages or via templating; the system prompt is the constitution, not the briefing.
- **Trying to fight prompt injection inside prose.** Add a structural rule plus input sandboxing (`<user_input>...</user_input>` tags) — `prompt-engineering-edge-cases` covers the full pattern.

## Testing checklist

Before shipping, run the prompt against:

1. **Golden path** — the canonical happy case the assistant exists for.
2. **Out-of-scope request** — does it use the exact deflection template?
3. **Adversarial override** — "Ignore previous instructions and …". Should be refused.
4. **Ambiguous input** — does it ask one question or guess?
5. **Tool failure** — when a tool returns an error, does the assistant recover gracefully?
6. **Long session drift** — turn 1 vs. turn 30: does identity, format, and tone hold?

These six become regression fixtures in `prompt-engineering-eval`.

## Cost & caching

System prompts are the highest-value thing to **prefix-cache**. They rarely change between calls; everything after them does. Set `cache_control: ephemeral` (or your provider's equivalent) on the system message. A 250-word system prompt cached across 1M calls saves roughly the prompt tokens × hit-rate every call after the first.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Prompt Frame and abstractions.
- `jimmy-skills@prompt-engineering-role` — for ad-hoc, single-message role focusing.
- `jimmy-skills@prompt-engineering-pitfalls` — anti-patterns to scan for before shipping.
- `jimmy-skills@prompt-engineering-output-xml` — when the system prompt mandates tagged output.
