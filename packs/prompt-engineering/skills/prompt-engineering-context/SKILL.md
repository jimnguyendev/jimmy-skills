---
name: prompt-engineering-context
description: "Engineer the context window: what to include, what to summarize, what to retrieve. Covers RAG, embeddings, chunking, ordering effects, summarization strategies, tool/function calling, and the Model Context Protocol (MCP). Use when the model needs information it didn't train on, or when long inputs are degrading quality."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Vector DB, embedding model, and MCP-server choices are independent of these patterns. Provider docs are authoritative for context-window limits and tool-calling APIs."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# Context Engineering

> **Disclaimer.** Provider-specific notes (context-window sizes, tool-calling APIs, MCP support) reflect the patterns at time of writing. Verify current limits and APIs in the provider's documentation before sizing a system.

LLMs are stateless. Every call starts from zero. What the model "knows" about your task is exactly what's in the context window — system prompt, conversation history, retrieved documents, tool results, the user message. Context engineering is the discipline of getting the right information into that window and nothing else.

## The context window is a budget

You are spending tokens on:

```text
[ system prompt ] [ persona / role ] [ few-shot examples ]
[ retrieved knowledge ] [ conversation history ]
[ user query ] [ scratchpad / reasoning ] [ output ]
```

Every token costs money, latency, and (past a point) quality — long contexts degrade attention to detail and amplify the lost-in-the-middle effect. Treat the window like RAM: relevant data only, evicted when stale.

## RAG (Retrieval-Augmented Generation)

When the model needs information it didn't train on (your docs, your runbooks, customer history), retrieve relevant pieces and inject them into the prompt at call time.

The pipeline:

```text
ingest:  documents → chunk → embed → store in vector DB
query:   user_question → embed → top_k similarity → assemble prompt
answer:  call_model(system + retrieved_chunks + user_question)
```

Three knobs that matter:

1. **Chunk size.** 200–800 tokens is the usable range. Too small loses context; too large drowns relevance scores. Match chunk to natural unit (a function, a paragraph, a Q&A pair).
2. **Top-k.** Start with k=5–10. Reranking afterwards (with a cross-encoder or a small LLM) typically beats raising k.
3. **Hybrid search.** Combine vector similarity with keyword (BM25). Vector handles semantics; keyword handles "exact match this product code." Most production RAG uses both.

## Embeddings

Text → vector. The vector encodes semantic meaning so similar concepts cluster.

- Pick an embedding model and pin the version. Re-embedding 1M docs because the model upgraded is painful.
- Match retrieval-side and ingest-side embeddings. Same model, same version.
- Cache embeddings of frequent queries.
- For multilingual, choose an embedding model trained on your languages.

## Chunking that works

- **Structure-aware** beats fixed-size: split on headings, function boundaries, or sentence groups, not byte counts.
- **Overlap** of 10–20% prevents losing facts that straddle chunk boundaries.
- **Metadata on every chunk:** source URL, section, modified date, author. The model uses these to cite; you use them to filter.
- **Reject too-small chunks.** A 30-token chunk almost always loses context.

## Ordering inside the context

Models attend most strongly to the **start** and the **end** of long contexts (the lost-in-the-middle effect). When you have a budget of slots, place the highest-stakes content at the edges:

```text
[ system prompt ]                          ← strong attention
[ critical retrieved chunks ]
[ ... less critical chunks ... ]           ← weak attention (lost in the middle)
[ recent conversation summary ]
[ user query ]                             ← strong attention
```

Two practical implications:

1. Put the most-relevant retrieved chunk **last** in the retrieval block, just before the user query.
2. Repeat the most important constraint at both ends if the prompt is long.

## Conversation history

Long conversations exceed the budget. Three strategies, often combined:

- **Window.** Keep last N turns verbatim; drop the rest. Cheap, loses long-term memory.
- **Summarize-and-roll.** Periodically replace the oldest turns with a short summary. Preserves decisions, facts, and tone. Aim for ~20% the original token count.
- **Retrieve-from-history.** Embed past turns and treat them like RAG. Useful for assistants with weeks of conversation.

A useful default: keep the last 6 turns verbatim, summarize anything older, and re-summarize when the summary itself exceeds 500 tokens.

## Tool / Function calling

When the model needs **fresh** or **interactive** data, give it tools instead of more retrieved documents:

```text
get_account_status(account_id) → {plan, mrr, status}
search_kb(query) → list of articles
open_ticket(subject, body, severity) → ticket_id
```

The model emits structured tool calls; your code executes; you feed the result back. Three rules:

- **Schema-first.** Tool input/output is just a JSON schema (`prompt-engineering-output-json`).
- **Few, well-named tools.** Twenty similar tools confuse the model. Five with clear names work better.
- **Idempotent reads, confirmed writes.** Reads can be retried freely. Writes (delete, send, charge) require an explicit confirmation step.

## Model Context Protocol (MCP)

MCP standardizes how a client (an LLM agent) discovers and calls tools / resources / prompts hosted on an MCP server. Useful when:

- You have multiple agents that need the same tools.
- Tool authors are different from agent authors.
- You want tools that work across providers without rewriting integrations.

Patterns:

- Treat each MCP server as a **bounded capability** (database access, file system, vector DB, calendar). Don't expose generic shell.
- Cache the tool list — it rarely changes between calls; loading it every turn wastes tokens.
- For high-volume routes, bypass MCP and call the underlying API directly; reserve MCP for agent-driven exploration.

## Summarization

Summarize when:

- Conversation history exceeds the budget.
- You're caching long-form context that downstream prompts reference.
- The user wants a brief over a corpus.

Anchor the summary to **what the next step needs**:

```text
Summarize the conversation above into ≤300 tokens, preserving:
- Decisions made (each as an imperative sentence).
- Facts about the user (preferences, identity, account).
- Open questions / pending actions.
- Tone the user prefers.
Discard small talk.
```

Summaries for downstream models are different from summaries for human readers. Optimize for the consumer.

## Keeping the system prompt small

The system prompt is the most-frequently-loaded part of the context. Keep it under ~500 words. Push detail to:

- Retrieved knowledge (RAG): policies, product info, edge-case rules.
- Tool descriptions: behavior of each tool.
- Per-session task context: the user's specific goal.

## Anti-patterns

- **Pasting whole documents** when you only need three paragraphs. Chunk and retrieve.
- **No retrieval, just a giant prompt.** Quality degrades past ~32k tokens on most models even when the window allows more.
- **Retrieval with no metadata.** You can't filter by recency, source, or tenant.
- **Top-k = 50 because "more is better."** It dilutes the signal and inflates cost.
- **Re-embedding mismatched models.** Ingest with model A, query with model B → near-random retrieval.
- **Long conversation history kept verbatim forever.** Summarize, or you'll hit limits at the worst time.
- **Tools that are too generic.** A `run_sql(query)` tool is a footgun; a `get_customer(id)` tool is safe.
- **Trusting the model to remember "what we discussed earlier."** It only remembers what's currently in context.

## Quick template

```text
[ system prompt — identity, rules, format          ]   ← cached prefix
[ tool definitions — JSON schemas                  ]   ← cached prefix
[ few-shot examples                                ]   ← cached prefix
[ retrieved chunks (top-k, reranked, with metadata)]   ← per-call
[ conversation summary (older turns)               ]   ← per-call
[ recent turns verbatim (last N)                   ]   ← per-call
[ user query                                       ]   ← per-call
```

The cached prefix is often >80% of the prompt; only the last three sections vary per call. That's the architecture cost-engineering depends on.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — context is the second slot of the Prompt Frame.
- `jimmy-skills@prompt-engineering-system-prompt` — keep the constitution lean; push detail here.
- `jimmy-skills@prompt-engineering-chain` — chains often have a retrieve step.
- `jimmy-skills@prompt-engineering-cost` — caching the stable prefix is the highest-leverage cost lever.
- `jimmy-skills@prompt-engineering-agent` — agents use tools and MCP in a loop.
