---
name: prompt-engineering-output-json
description: "Design, emit, validate, and parse JSON outputs from LLMs. Schema-first prompting, null-handling, enum constraints, array verification, self-validation step, and ready-to-use Pydantic / Zod parsers. Use whenever downstream code (an API, a pipeline, a tool dispatcher) consumes the model's output."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. When the provider supports a native JSON / structured-output mode (Anthropic tool-use, OpenAI response_format=json_schema, Gemini responseSchema), prefer that — these patterns still apply on top."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep Bash
---

# JSON Output Contracts

> **Disclaimer.** Provider-specific notes (Anthropic tool-use, OpenAI `response_format`, Gemini `responseSchema`) reflect the patterns at time of writing. The provider's official documentation remains authoritative — verify schema-mode behavior against current docs before shipping.

JSON is the default format whenever code consumes the model's output. This skill turns "return JSON" from a wish into a contract: schema-first, null-tolerant, parser-checked, and resilient to the three failure modes that account for almost all production breakages.

## When to use

- An API endpoint, queue worker, or pipeline parses the model output.
- You need typed fields, enums, or arrays of records.
- A tool-dispatcher reads the model's choice and arguments.
- You want regression-testable structure (the schema is the contract — fixtures assert against it).

If the output is for human reading, prefer markdown via `prompt-engineering-output-structured`. If you are running on Claude with long, nested document inputs, also consider `prompt-engineering-output-xml`.

## Provider-native modes first

Before hand-rolling, check whether the provider gives you a structured-output mode:

- **Anthropic** — use a tool definition with an `input_schema`. Even if you don't actually execute the tool, this is the most reliable way to constrain Claude's output to a JSON Schema.
- **OpenAI** — `response_format: { type: "json_schema", json_schema: {...} }`.
- **Gemini** — `responseMimeType: "application/json"` plus `responseSchema`.

Use the patterns below when the provider lacks native support, when you need finer-grained behavior, or as the *internal contract* you also pass to the native mode.

## Schema-first prompting

Define the schema before the prose.

```text
<output>
Return ONLY valid JSON matching this schema. No markdown. No commentary.

{
  "ticket_id": "string",
  "category": "billing" | "technical" | "account" | "other",
  "severity": 1 | 2 | 3 | 4,
  "summary": "string (<= 240 chars)",
  "next_action": {
    "kind": "reply" | "escalate" | "close",
    "owner": "string | null"
  },
  "confidence": "number (0..1)",
  "tags": "string[] (<= 5 items)"
}

Rules:
- Use null for unknown fields. Do NOT invent values.
- "category" and "severity" must be one of the listed values.
- "tags" must be lowercase, kebab-case, and contain no duplicates.

Before responding, verify that the JSON parses and every required field
is present.
</output>
```

Why this works:

- **TypeScript-style annotations** are read more reliably than prose ("a number between 0 and 1").
- **Enum constraints inline** ("billing" | "technical" | …) reduce hallucinated categories.
- **Length constraints** ("<= 240 chars", "<= 5 items") prevent runaway outputs.
- **Self-validation directive** at the end catches the easy mistakes before the model commits to its first token of output.

## Null over invent

State this rule **explicitly**:

> Use `null` for any field whose value cannot be determined from the input. Do NOT invent.

LLMs default to filling every field with *something* when asked. The null-over-invent rule is the single highest-leverage line in any extraction prompt and turns half of all "hallucinated field" bugs into easily-detectable missing data.

## The three failure modes

Almost every JSON parse error in production traces back to one of these:

1. **Markdown wrapping.** The model wraps output in ```` ```json ```` fences.
   - **Prompt fix:** "Return ONLY valid JSON. No markdown. No code fences."
   - **Parser fix:** strip leading/trailing fences before parsing (defense in depth).

2. **Trailing commas.** Especially in lists of objects.
   - **Prompt fix:** "JSON must be strictly RFC 8259-compliant; no trailing commas."
   - **Parser fix:** if the language allows (JSON5, json_repair), retry with a lenient parser before failing.

3. **Unescaped quotes / control chars in strings.** User content with `"` or newlines.
   - **Prompt fix:** "Escape all double quotes and newlines inside string values."
   - **Parser fix:** when you control the schema, prefer fields that won't carry raw user prose; or run a repair pass.

A good rule: **prompt to prevent, parse to recover.** Both, not either.

## Patterns

### Array length verification

When the model emits an array, it is easy to truncate. Add a count field the model must match:

```json
{
  "items_count": "integer",
  "items": "Item[]"
}
```

Then validate `items_count == len(items)` after parse. If the model lies, you catch it immediately.

### Optional fields

Use `null` rather than omitting the key, and rather than empty strings (which encode "I knew but it was blank" and lead to hallucination).

```json
{ "phone": "string | null", "email": "string | null" }
```

### Discriminated unions

When a field can be one of several shapes, use an explicit `kind` discriminator:

```json
{
  "action": {
    "kind": "send_email" | "open_ticket" | "do_nothing",
    "payload": "EmailPayload | TicketPayload | null"
  }
}
```

Discriminated unions parse cleanly with Pydantic (`Annotated[..., Field(discriminator="kind")]`) and Zod (`z.discriminatedUnion`).

### Confidence + sources

For any extracted or judged value, add:

```json
{ "value": "...", "confidence": "number (0..1)", "evidence": "string" }
```

This makes downstream filtering trivial (drop where `confidence < 0.7`) and makes regressions debuggable.

## Worked end-to-end example (Python + Pydantic)

```python
from typing import Literal, Optional
from pydantic import BaseModel, Field, ValidationError
import json, re

class NextAction(BaseModel):
    kind: Literal["reply", "escalate", "close"]
    owner: Optional[str] = None

class Ticket(BaseModel):
    ticket_id: str
    category: Literal["billing", "technical", "account", "other"]
    severity: Literal[1, 2, 3, 4]
    summary: str = Field(max_length=240)
    next_action: NextAction
    confidence: float = Field(ge=0.0, le=1.0)
    tags: list[str] = Field(max_length=5)

FENCE = re.compile(r"^```(?:json)?\s*|\s*```$", re.MULTILINE)

def parse(raw: str) -> Ticket:
    cleaned = FENCE.sub("", raw).strip()
    try:
        return Ticket.model_validate_json(cleaned)
    except ValidationError as e:
        # Optionally: one repair retry with the same model, passing e.errors() back.
        raise
```

Equivalent Zod sketch:

```ts
import { z } from "zod";

const Ticket = z.object({
  ticket_id: z.string(),
  category: z.enum(["billing", "technical", "account", "other"]),
  severity: z.union([z.literal(1), z.literal(2), z.literal(3), z.literal(4)]),
  summary: z.string().max(240),
  next_action: z.object({
    kind: z.enum(["reply", "escalate", "close"]),
    owner: z.string().nullable(),
  }),
  confidence: z.number().min(0).max(1),
  tags: z.array(z.string()).max(5),
});
```

Both schemas are also the input you'd pass to a provider's native structured-output mode — single source of truth.

## Self-validation step

The final line of the prompt should ask the model to **check its own output before emitting**:

> Before responding, verify: (1) the JSON parses, (2) every required field is present, (3) all enum values are from the allowed set, (4) array lengths match their count fields. If any check fails, fix and re-emit.

This is cheap (a few extra tokens of "thinking" before the first output token) and catches a meaningful fraction of malformed outputs without needing a repair retry.

## Repair, don't retry blindly

When parsing fails, prefer **targeted repair** over full retry:

1. Strip markdown fences.
2. Try a lenient parser (`json_repair` in Python, `JSON5` in JS) before failing.
3. If still failing, send the model back its own output plus the parser error message and ask it to return *only* corrected JSON.

A blind retry costs the same input tokens twice with no extra signal. A repair retry usually fixes on the first attempt.

## Cost & caching

- **Stable schema → prefix cache.** The schema rarely changes between calls. Put it in the system prompt or the cached prefix.
- **Prefer short keys** when output volume matters (`cat` vs. `category` saves ~5 tokens per record × N records). Trade off readability.
- **Bound `max_tokens`.** Compute from the largest plausible output; prevents runaway cost when the model loops.

## Anti-patterns

- "Return JSON" with no schema. You will get differently-shaped output across calls.
- Empty strings for missing values. Forces downstream code to special-case `""`; use `null`.
- Unbounded arrays. Without length caps, the model will hallucinate items to "fill out" the response.
- Putting prose alongside JSON ("Here is the JSON: …" + "Let me know if you need more"). Either say "JSON only" or use a wrapper field for commentary.
- Not validating after parse. The schema is your test surface — if you skip validation, you don't have a contract, you have hope.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — Output Contract abstraction.
- `jimmy-skills@prompt-engineering-output-xml` — when nested document structure is the bottleneck.
- `jimmy-skills@prompt-engineering-edge-cases` — empty / long / ambiguous input handlers.
- `jimmy-skills@prompt-engineering-eval` — golden-set fixtures for schema regressions.
