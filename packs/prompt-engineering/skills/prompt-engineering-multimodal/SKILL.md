---
name: prompt-engineering-multimodal
description: "Prompt patterns for image, audio, and video inputs/outputs. Covers structured image analysis, screenshot/document extraction, image generation prompts (subject + style + composition + lighting + mood + specs), audio/video transcription with timestamps, and the OCR-trust rule. Use when the input or output isn't text."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Image, audio, and video capabilities differ across providers — verify supported formats, max dimensions, and frame-rate handling in provider docs."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Multimodal Prompting

> **Disclaimer.** Multimodal capabilities and limits (max image size, audio duration, frame extraction) vary by provider and model. The provider's documentation is authoritative.

The model can see images, hear audio, watch video. What it can't do is **decide what matters** — that's the prompt's job. Without guidance, a multimodal model describes colors, layout, irrelevant background; with guidance, it focuses on the conversion rate, the error indicator, the data anomaly you actually care about.

## When to use

- Input is an image, screenshot, document scan, audio file, or video.
- Output is an image (generation prompts).
- Mixed input: text + image (debug a UI bug + screenshot).

## Image analysis — the structured pattern

Don't ask "what do you see?" Ask for specific extractions:

```text
You will be given a screenshot of <surface>. Extract the following.

<task>
Return JSON:
{
  "primary_metric": { "label": "...", "value": "...", "delta_pct": <number | null> },
  "anomalies": [
    { "region": "top-right | bottom-left | ...", "description": "...", "severity": "low|med|high" }
  ],
  "ui_state": "loading | empty | error | normal",
  "confidence": <0..1>
}
</task>

<constraints>
- Use spatial language (top-right, bottom-left, header, footer) when locating elements.
- If the screenshot is blurry, low-resolution, or partially cut off: set confidence < 0.5
  and list the impacted fields in "limitations".
- NEVER guess at illegible text — return null and note it.
</constraints>
```

Why it works: structure forces focus on the fields you specified, not on whatever the model finds visually striking.

## Document & screenshot extraction

For receipts, invoices, forms, tables in screenshots:

```text
Extract the structured data into JSON matching the schema. Use null when a
field is not visible or unreadable. Do NOT infer values not in the document.

Schema:
{
  "vendor": "string | null",
  "date": "YYYY-MM-DD | null",
  "currency": "ISO 4217 | null",
  "total": "number | null",
  "line_items": [{ "description": "string", "qty": "number", "unit_price": "number" }],
  "ocr_confidence": "high | medium | low"
}
```

Two non-negotiables:

1. **Verify** any extracted number that's load-bearing (matches against an expected total, sums cross-foot, etc.).
2. **OCR is not perfect.** Especially handwriting, decorative fonts, low-light photos. Track an `ocr_confidence` field and gate downstream use.

## Image generation prompts

Generation prompts are a different shape — they're descriptions, not instructions. Six recurring slots:

```text
<subject + action>     "A wise elderly wizard reading an ancient tome"
<setting>              "in a tower library at sunset"
<style>                "watercolor with detailed line work"
<composition>          "medium shot, rule of thirds, subject left"
<lighting + mood>      "soft golden-hour light through stained glass; serene, contemplative"
<technical specs>      "16:9, sharp focus on the book, shallow DOF on background"
```

Plus an optional **negative prompt** (what to exclude): "no text, no watermarks, no modern objects."

Generic prompts produce generic images. Specific slots produce art. The pattern works across DALL·E, Stable Diffusion, Midjourney, Imagen, Firefly — the **slot vocabulary differs**, but the discipline of filling all six is universal.

## Audio & video

For transcription, ask for structure:

```text
Transcribe the audio. Output JSON with timestamps:

{
  "language": "ISO 639-1",
  "speakers": [{ "id": "S1", "label": "Interviewer" }, ...],
  "segments": [
    { "start": 0.0, "end": 4.3, "speaker": "S1", "text": "..." }
  ],
  "summary": "≤ 3 sentences"
}

Constraints:
- Diarize speakers if more than one is audible.
- For unclear sections, mark "[inaudible]"; do not invent words.
- Time codes in seconds, two decimal places.
```

For video analysis, give the model **what to look for, frame-by-frame**:

```text
Analyze the video at 1fps sampling. For each second, return:
- Whether the user appears to be on the target screen (yes/no/unclear).
- Any visible error indicators.
- Notable state changes.
Aggregate into a timeline at the end.
```

Frame budgets matter — multimodal calls are expensive. Sample at the lowest rate that captures the events you care about.

## Mixed input: text + image

This is where multimodal pays off. Give the model **the question and the artifact** together:

```text
The screenshot below shows my checkout page after the latest deploy.
The orange CTA button no longer fires the analytics event we expect.

<screenshot/>

Here's the relevant JS (omitted in this template — paste yours):

[code block with the JS handler]

Tell me:
1. What in the screenshot looks unusual (1–3 observations).
2. What in the JS could explain it (cite line numbers).
3. The single most likely root cause and a one-line fix.
```

This is also how you debug visual regressions, evaluate design options, or compare a Figma frame to a built UI.

## Spatial language

Models are better at locations described with **named regions** than with pixel coordinates:

| Use | Don't use |
|---|---|
| "top-right", "footer", "navigation bar" | "around 1200,80" |
| "the third button from the left" | "x=420" |
| "below the headline, above the form" | "y=350 to y=500" |

Pixel coordinates are unreliable across resolutions and DPRs.

## Verification rules

For any high-stakes extraction:

- Re-prompt with a verification frame: *"You extracted X. Re-read the document and confirm. If the document doesn't show X, return CONTRADICTION with what it does show."*
- Cross-check totals (line items sum to total).
- For dates and IDs, return `as_written` plus `parsed_iso` so a mismatch is visible.

The OCR-trust rule: **never act on extracted data that hasn't been verified by an independent rule, a human, or a second pass.**

## Anti-patterns

- **"What do you see?"** Vague → vague. Always specify the fields.
- **Trusting OCR on handwriting / low-res.** Always include a confidence field; gate downstream use.
- **Asking the model to identify specific people.** Many providers refuse; many will hallucinate. Treat people as "person 1, person 2."
- **Generic image-gen prompts.** "A nice image of a forest" → averaged-out clip-art. Fill all six slots.
- **Sampling video at 30fps when 1fps is enough.** Pure cost.
- **Pixel coordinates.** Brittle. Use named regions.
- **Ignoring image-size limits.** Many providers downscale silently; small text becomes unreadable. Check dimensions vs. provider limit and crop locally if needed.
- **Treating multimodal output as ground truth.** It isn't. Validate.

## Quick templates

**Image analysis:**

```text
<role>...</role>
<task>Extract the following from the screenshot.</task>
<image/>
<output>
JSON: { ...specific fields with types and nulls allowed... }
</output>
<constraints>
- Spatial language only.
- Null over invent for illegible / not-present fields.
- Return confidence 0..1.
</constraints>
```

**Image generation:**

```text
Subject: <subject + action>
Setting: <where, when>
Style: <medium / artist / era>
Composition: <shot type, framing, focal subject>
Lighting & Mood: <light source + emotional tone>
Technical: <aspect ratio, focus, detail level>
Negative: <what to exclude>
```

**Audio transcription:**

```text
Transcribe with timestamps and speaker diarization. Return JSON
with segments, language, summary. Mark [inaudible] for unclear sections.
```

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Prompt Frame.
- `jimmy-skills@prompt-engineering-output-json` — schema for extracted data.
- `jimmy-skills@prompt-engineering-edge-cases` — verification rules and degraded-input handling.
- `jimmy-skills@prompt-engineering-cost` — sampling-rate and resolution-vs-cost tradeoffs.
