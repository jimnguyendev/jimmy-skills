## Prompt Engineering Pack

Production-grade prompt design — patterns, contracts, and economics for building reliable LLM features and agents.

The pack is organized around three reusable abstractions every skill agrees on:

1. **Prompt Frame** — a 7-slot canonical template (`role`, `context`, `task`, `inputs`, `constraints`, `reasoning`, `output`). Skills compose by filling slots, not by rewriting prose.
2. **Output Contract** — every structured output ships with a schema, parser, positive + negative example, and known failure modes.
3. **Cost Profile** — every template carries `{model_tier, est_input_tokens, est_output_tokens, cache_strategy}` so routers can pick automatically.

### Routing

- Start with `prompt-engineering-core` for the Prompt Frame and "which skill do I need" map.
- Use `prompt-engineering-system-prompt` when designing a persistent agent identity (vs. a one-shot user message).
- Use `prompt-engineering-role` to focus a single message on a specific expertise lens.
- Use `prompt-engineering-output-json` when downstream code parses the output. Use `prompt-engineering-output-xml` when running on Claude with long context or nested document structure.
- Use `prompt-engineering-pitfalls` as a pre-flight checklist before shipping any prompt.

### Foundations

| Skill | When to use |
|---|---|
| `prompt-engineering-core` | Foundation. Prompt Frame, anatomy, and routing map. Read first. |
| `prompt-engineering-system-prompt` | Designing a persistent system prompt for an agent or product. Five-part + layered architecture, <500 words. |
| `prompt-engineering-role` | Activating expert behavior in a single message. Role stack, compound/situational/perspective roles, anti-patterns. |
| `prompt-engineering-output-json` | Schema-first JSON outputs with null-handling, enum constraints, self-validation, and parser snippets. |
| `prompt-engineering-output-xml` | XML-tagged prompts for Claude long-context and nested documents. |
| `prompt-engineering-pitfalls` | Nine anti-patterns and a debug checklist. Use before shipping. |

### Techniques

| Skill | When to use |
|---|---|
| `prompt-engineering-few-shot` | Show, don't describe. The 2–5 rule, diversity-over-quantity, negative examples, edge-case demos, ordering. |
| `prompt-engineering-reasoning` | Chain-of-thought for math, debugging, planning. Zero-shot CoT, BREAK template, scratchpad pattern, self-consistency, and when CoT is just overhead. |
| `prompt-engineering-refine` | Write → Test → Analyze → Improve loop. Single-variable iteration, symptom→fix table, version log, stop rule. |
| `prompt-engineering-edge-cases` | Defensive prompt design. Input/Domain/Adversarial edge cases, prompt-injection layered defense, graceful degradation. |
| `prompt-engineering-output-structured` | Markdown structure for human-rendered output. Lists, tables, headers, emphasis directives, conditional formatting. |
| `prompt-engineering-output-yaml` | YAML for k8s, Compose, CI configs, runbooks. Norway problem, anchors, multi-document, safe loading. |

### Production runtime

| Skill | When to use |
|---|---|
| `prompt-engineering-chain` | Decompose tasks into specialized steps (sequential / parallel / conditional / iterative). The Extract → Transform → Generate baseline. |
| `prompt-engineering-context` | RAG, embeddings, chunking, ordering effects, summarization, tools, and MCP. Engineering the context window as a budget. |
| `prompt-engineering-cost` | Model routing, prefix caching, response caching, token compression, batch APIs, streaming, the quality/cost/latency triangle. |
| `prompt-engineering-eval` | Golden fixtures, scoring strategies (rule-based / reference / LLM-as-judge), CI integration, drift monitoring. |
| `prompt-engineering-agent` | Agent loop (Plan → Execute → Observe → Adapt), tool-use prompt format, recovery prompts, step budgets, multi-agent. |
| `prompt-engineering-multimodal` | Image / audio / video prompting. Structured analysis, image-gen six-slot pattern, OCR-trust rule. |

### Domain templates

Ready-to-paste Prompt-Frame templates per domain. Drop in inputs, ship.

| Skill | Templates included |
|---|---|
| `prompt-engineering-domain-coding` | PR review, debug, code-gen, architecture, test-gen (AAA), refactor proposal. |
| `prompt-engineering-domain-writing` | Blog, marketing copy, brand-voice extraction, three-phase editing, outline-first long-form, SEO. |
| `prompt-engineering-domain-education` | Adaptive tutor, Socratic mode, lesson plan, quiz, learning path, flashcards. |
| `prompt-engineering-domain-business` | Email, decision frameworks (SWOT/weighted/pre-mortem), meeting agenda, OKR, strategic plan, SOP. |
| `prompt-engineering-domain-creative` | Image-gen six slots, three-act story, character design, dialogue, worldbuilding, song, sound, game design, 10/5/3/1 brainstorm. |
| `prompt-engineering-domain-research` | Paper summary, lit synthesis, PESTLE, 5 Whys, gap analysis, CRAAP, methodology design. |
| `prompt-engineering-domain-support` | Ticket triage, KB-grounded reply, escalation policy, KB authoring, macros. |
| `prompt-engineering-domain-devops` | Incident triage, blameless postmortem, runbook, IaC review, CI design, on-call comms. |

### Disclaimer

Techniques here are distilled from the prompts.chat *Interactive Book of Prompting* and Anthropic's published prompt-engineering guidance. Provider docs remain authoritative for model-specific behavior.
