---
name: prompt-engineering-output-structured
description: "Shape human-readable outputs with markdown structure: lists, tables, headers, emphasis directives (MUST/NEVER), typed entities, and conditional formatting. Use when the output is read by a human or rendered in a UI — for code-consumed outputs, prefer prompt-engineering-output-json or output-xml."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Markdown rendering varies by surface (chat, IDE, web); confirm rendering before assuming feature support (tables, callouts, footnotes)."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Structured Markdown Outputs

When a human reads the output, structure beats prose. Headers let readers scan, tables let them compare, lists let them act. This skill is about shaping that structure so it's predictable across calls.

For code-consumed outputs (parsing, validation, downstream pipelines), use `prompt-engineering-output-json` or `prompt-engineering-output-xml` instead. Markdown is fragile to parse; treat it as a presentation layer, not a contract.

## When to use

- Output renders directly to a human (chat UI, doc page, README, email).
- The reader needs to **scan** (headers) or **compare** (tables) more than **read straight through**.
- A downstream renderer supports markdown (Slack, GitHub, Notion, most chat UIs).
- The output mixes prose with enumerated items, decisions, or trade-off analysis.

## Choosing the right structure

| Need | Use | Notes |
|---|---|---|
| Sequential steps | Numbered list | Specify count up front. |
| Unordered set of items | Bullet list | Cap the count. |
| Compare items across attributes | Table | 4–6 columns max for readability. |
| Document with sections | Headers (`##`, `###`) | List section names in desired order. |
| Categorized blocks | Headers + lists per section | Limit to 3–5 sections. |
| Decision with trade-offs | Header + table + summary line | Force the model to commit to a recommendation. |
| Long answer with action items | Summary → details → next steps | Lead with the verdict. |

## Lists

Specify three things or the model improvises:

1. **Count.** "Exactly 5 items." Otherwise you get 3 to 9.
2. **Whether each item gets an explanation.** "One sentence per item, or just the label."
3. **Ordering rule.** "Most important first." / "Alphabetical." / "Chronological."

Bad: *"List the trade-offs."*
Good: *"List exactly 4 trade-offs, most important first, with a one-sentence explanation each."*

## Tables

Define the columns explicitly. Models will invent columns and merge cells if you don't pin them.

```text
Return a markdown table with EXACTLY these columns and order:

| Option | Pros | Cons | Recommended for |

- 4–6 rows.
- One option per row.
- "Pros" / "Cons" must be a comma-separated list, no nested bullets.
- "Recommended for" must be a single noun phrase under 6 words.
```

When rendering targets are weak (Slack tables look ugly, some chat UIs don't render at all), fall back to definition lists or repeated headers.

## Headers

Specify the section names in desired order:

```text
Return a markdown document with these H2 sections, in this order:

## Summary
## Findings
## Recommendation
## Open Questions
```

Two rules:

1. **Don't let the model invent sections.** Explicit list, in order.
2. **Cap depth at H3.** Deeper hierarchies hurt scannability and most renderers stop styling there.

## Emphasis directives

Uppercase keywords act as strong constraints on model behavior. Use them to communicate priority, not for visual emphasis in the output.

| Directive | Effect |
|---|---|
| **MUST** | Hard requirement, included always. |
| **NEVER** | Hard prohibition. |
| **ALWAYS** | Like MUST, used for behaviors. |
| **DO NOT** | Equivalent to NEVER. |
| **ONLY** | Restricts to a closed set. |
| **IMPORTANT** | Soft priority — use sparingly to avoid dilution. |

The dilution rule: if every line is IMPORTANT, none of them are. Reserve uppercase directives for the 2–4 things that genuinely break the output if violated.

## Typed entities

For NER-style outputs in markdown, label inline:

```text
For each named entity, prefix with its type in brackets:
[PERSON], [ORG], [LOCATION], [DATE], [MONEY], [PRODUCT].

Example: "[PERSON]Alice[/PERSON] joined [ORG]Acme[/ORG] in [DATE]2023[/DATE]."
```

This is a degenerate XML and parses with a simple regex if needed downstream — but it stays readable in chat UIs that don't render tags.

## Conditional formatting

Vary structure based on input. State the conditions explicitly:

```text
- If <input> contains exactly one option: return a single paragraph.
- If <input> contains 2–4 options: return a comparison table.
- If <input> contains 5+ options: return a categorized list grouped by <criterion>.
```

The model handles conditional formatting reliably when the conditions are mutually exclusive and exhaustive.

## Quick patterns

### TL;DR + details

```text
**TL;DR.** <one sentence verdict>

## Why
<2–3 sentences>

## Details
<bullet list, ≤5 items>
```

The TL;DR-first pattern dominates chat-rendered outputs because users skim. Always commit to a verdict in the first line.

### Decision summary

```text
## Recommendation
<one sentence with verdict>

| Option | Score | Key reason |
| --- | --- | --- |
| A | 8/10 | ... |
| B | 6/10 | ... |
| C | 4/10 | ... |

## Trade-offs
- ...

## Risks
- ...
```

### Code review reply

```text
**Verdict:** approve | request_changes | needs_discussion
**Summary:** <2 sentences>

## Comments

- 🔴 `path/to/file.go:42` — <critical issue>
- 🟡 `path/to/file.go:80` — <important suggestion>
- 🟢 `path/to/file.go:115` — <nit>
```

(Domain-specific templates live in `prompt-engineering-domain-coding`.)

## Anti-patterns

- **Nested-list overuse.** Three levels of bullets become noise. Promote to headers.
- **Mixing numbered and bulleted lists for the same set.** Inconsistent rendering across runs.
- **Decorative emojis in production output.** Unless explicitly part of the spec, drop them — they bloat tokens and look unprofessional in many surfaces.
- **Tables in surfaces that don't render them.** Test the actual surface (Slack, MS Teams, terminal) before shipping.
- **Headers without content rules.** "Use ## sections" with no list of section names → the model invents.
- **Uppercase EVERYWHERE.** Dilutes signal. Reserve for the 2–4 lines that matter.
- **Markdown as a parser contract.** If code consumes it, switch to JSON/XML.

## Quick template

```text
<output>
Return a markdown document with this exact structure:

**TL;DR.** <one sentence>

## Findings
- Exactly N bullets, one sentence each, most important first.

## Comparison
| Column A | Column B | Column C |
| ---      | ---      | ---      |
| 4 rows of data, one per option |

## Recommendation
<one paragraph, ending with the chosen option in bold>

Rules:
- MUST start with the TL;DR line.
- NEVER add sections beyond the three above.
- ONLY use H2 headers; no H3 or deeper.
</output>
```

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Prompt Frame and Output Contract.
- `jimmy-skills@prompt-engineering-output-json` — when code consumes the output.
- `jimmy-skills@prompt-engineering-output-xml` — mixed structure + prose for Claude.
- `jimmy-skills@prompt-engineering-output-yaml` — config-style outputs.
