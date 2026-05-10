---
name: prompt-engineering-domain-writing
description: "Templates for content writing: blog posts, marketing copy, brand-voice extraction, three-phase editing (draft → refine → polish), and SEO-aware structure. Each template specifies audience, tone, length, and section structure as required slots — vagueness is the #1 cause of generic copy."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Pairs with brand-voice samples in <context>."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Domain — Writing & Content

Templates for producing copy that matches a brand and a goal — not generic clip-art prose. The single biggest lever for content quality is **specifying audience, tone, and length up front**; almost every "AI writing sounds AI" complaint traces to one of those being missing.

## Template — blog post

```text
<role>
You are a content writer who has covered {{topic}} for {{audience}}.
</role>

<task>Write a blog post on {{specific angle}}.</task>

<inputs>
<audience>{{role + experience level + the question they came to answer}}</audience>
<tone>{{2–3 adjectives}}</tone>
<keywords>{{3–5 phrases to use naturally}}</keywords>
<brand_voice_sample>{{paste 1–2 paragraphs of existing copy}}</brand_voice_sample>
<source_material>{{links, notes, research}}</source_material>
</inputs>

<constraints>
- Length: {{N}} words ± 10%.
- One sentence per line in the draft.
- H2 headers, no H3.
- Open with a hook (anecdote, contrarian claim, or surprising stat).
- Close with a CTA: {{specific action}}.
- Each section ends with a takeaway the reader can act on.
- Do NOT use these clichés: "in today's fast-paced world", "game-changer", "synergize".
</constraints>

<output>
# {{title}}

**TL;DR.** One-sentence summary.

(Hook paragraph)

## Section 1
...

## Section 2
...

## Section 3
...

## What to do next
(CTA)
</output>
```

## Template — marketing copy (benefit-first)

```text
<role>
You are a copywriter focused on conversion. You write benefits, not features.
</role>

<task>Write {{landing page hero | email | ad | one-pager}} for {{product}}.</task>

<inputs>
<audience>{{specific persona, current pain, current alternatives}}</audience>
<offer>{{product, price, hook}}</offer>
<features>{{technical capabilities — for context only, not the focus}}</features>
<proof>{{customer numbers, testimonials, awards}}</proof>
<channel>{{landing page | email | LinkedIn ad | etc.}}</channel>
</inputs>

<constraints>
- Lead with the outcome the user wants, not the feature you ship.
- Translate every feature into a benefit ("X enables Y so you can Z").
- Length: {{constraint per channel}}.
- No marketing jargon: "leverage", "unlock", "robust", "best-in-class".
- One CTA. Action-verb-first.
</constraints>

<output>
{{format per channel — see examples below}}
</output>
```

Channel examples:

- **Landing hero**: 6-word headline, 1-sentence sub, 3 bullets (each 6–8 words), CTA button text (≤4 words).
- **Cold email**: subject (≤7 words), 3 short paragraphs, signature, CTA.
- **LinkedIn ad**: headline (≤25 chars), description (≤30 words), CTA.

## Template — brand-voice extraction

When the brief says "match our tone" but you don't have a style guide:

```text
<role>
You are a senior copy lead analyzing voice patterns.
</role>

<task>Extract the brand voice from the samples below and produce a usable style sheet.</task>

<inputs>
<samples>
{{paste 3–5 representative pieces of existing copy}}
</samples>
</inputs>

<output>
## Voice in 3 sentences
A reader who saw new copy in this voice would say: "Sounds like {{brand}}."

## Sentence patterns
- Average sentence length: ...
- Typical opener: ...
- Typical closer: ...

## Vocabulary
- High-frequency words and phrases: ...
- Words to avoid: ...

## Rhetorical devices
Lists, contrasts, metaphors, second-person address, etc.

## Tone scale
| Axis | Position |
| Formal ↔ Casual | ... |
| Earnest ↔ Playful | ... |
| Direct ↔ Discursive | ... |
| Confident ↔ Hedged | ... |

## Anti-examples
A sentence that violates the voice and why.
</output>
```

This becomes the `<brand_voice_sample>` block for every future writing prompt — a once-off investment that pays per piece.

## Template — three-phase editing

A single "edit this" prompt produces uneven results. Split into three passes:

```text
PASS 1 — DRAFT EDIT
<role>You are an editor focused on structure.</role>
<task>Identify the 3 biggest structural issues in this draft. Do NOT rewrite.</task>
<output>List of 3 issues + the move that fixes each (cut, reorder, merge, split).</output>

PASS 2 — LINE EDIT
<role>You are an editor focused on clarity and flow.</role>
<task>Tighten this draft sentence-by-sentence. Preserve meaning, voice, and length within 5%.</task>
<output>The full edited draft + a short list of recurring patterns you fixed.</output>

PASS 3 — POLISH
<role>You are a proofreader.</role>
<task>Fix grammar, punctuation, and stylistic inconsistencies. Flag (do not "fix") factual claims that should be verified.</task>
<output>Final draft + list of factual claims to verify.</output>
```

Run each as a separate call. Don't combine — the model averages across phases and you lose the discipline.

## Template — outline-first long-form

For pieces over ~1500 words:

```text
STEP 1 — OUTLINE
Produce an H2-level outline (5–8 sections) with one bullet per section
describing the takeaway. Do NOT write the body.

STEP 2 — REVIEW
{{User reviews and edits the outline.}}

STEP 3 — DRAFT (per section)
For each H2: write the full section using the agreed bullet as the
takeaway. Length per section: {{N}} words.
```

This is `prompt-engineering-chain` applied to writing. It catches structure problems before you've written 2000 words you need to throw away.

## SEO-aware structure

When SEO matters, layer it onto the blog template:

```text
<seo>
Primary keyword: {{phrase}}
Secondary keywords: {{2–3 phrases}}
Search intent: informational | navigational | commercial | transactional
Meta description: ≤160 chars, include primary keyword, end with implicit promise.
H1 includes primary keyword.
First 100 words include primary keyword once, naturally.
H2s include secondary keywords where natural.
NEVER stuff keywords; readability wins ties.
</seo>
```

Modern search rewards readers, not crawlers — keyword fit + actual usefulness, not density.

## Anti-patterns

- **"Write something professional about X."** Vague → average. Specify audience, tone, length, channel.
- **No brand-voice block.** AI-generic prose is the default; brand-voice is opt-in.
- **One mega-prompt for draft + edit + SEO + polish.** Combined objectives degrade each. Use the chain.
- **Banned words inside the prompt.** A long blocklist often surfaces the words anyway. Show 1–2 anti-examples; trust the model on the rest.
- **Cliché checks at the end.** Specify them up front in `<constraints>` — much easier to prevent than fix.
- **Missing CTA.** Marketing copy without a single, clear, action-verb CTA underperforms.
- **Length specified vaguely** ("not too long"). Give a number ± a range.

## Cost & model routing

| Task | Tier |
|---|---|
| Outlines | cheap |
| First drafts | mid |
| Brand-voice extraction | mid (one-time) |
| Final polish on a high-stakes piece | frontier |
| Headline / subject-line generation (n=10) | mid with n=10 + pick |

## Cross-references

- `jimmy-skills@prompt-engineering-role` — voice control via role + style.
- `jimmy-skills@prompt-engineering-few-shot` — brand-voice samples as few-shot.
- `jimmy-skills@prompt-engineering-chain` — outline → draft → edit → polish.
- `jimmy-skills@prompt-engineering-output-structured` — section / TL;DR layouts.
