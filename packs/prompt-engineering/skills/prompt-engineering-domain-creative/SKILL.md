---
name: prompt-engineering-domain-creative
description: "Templates for creative work: image-generation prompts (subject + setting + style + composition + lighting + tech), three-act story structure, character design, dialogue with distinct voices, worldbuilding, song structure, sound design, game mechanics, and the 10/5/3/1 brainstorm. Constraints fuel creativity — vague prompts produce average art."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Image-gen prompts work across DALL·E / Stable Diffusion / Midjourney / Imagen with minor vocabulary tweaks."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Domain — Creative Arts

The contrarian rule of creative prompting: **constraints don't kill creativity, they cause it.** A blank-canvas prompt produces averaged-out clip art. Tight constraints force unexpected, specific output — which is what creative work actually wants.

## Template — image generation

The six-slot frame. Fill all six.

```text
<image_prompt>
Subject + action: {{specific noun + verb, not "a cat" but "a black cat
                   stretching on a windowsill"}}
Setting:          {{where, when, weather, era}}
Style:            {{medium / artist / movement / era. e.g. "watercolor,
                   loose linework, in the style of Hayao Miyazaki backgrounds"}}
Composition:      {{shot type, framing, focal subject, rule of thirds,
                   leading lines}}
Lighting + mood:  {{light source, color temperature, time of day,
                   emotional tone}}
Technical specs:  {{aspect ratio, focus depth, detail level}}

Negative:         {{what to exclude — text, watermarks, modern objects,
                   distortions}}
</image_prompt>
```

Vocabulary tips:

- **Style** — name a medium and 1–2 references. "Watercolor + Beatrix Potter" beats "nice illustration."
- **Composition** — borrow from photography: medium shot, two-shot, close-up, Dutch tilt, low angle.
- **Lighting** — golden hour, blue hour, rim light, overcast, candlelit, neon, chiaroscuro.
- **Mood** — pick 1–2 adjectives, no more. "Peaceful, contemplative" not "peaceful, contemplative, but also adventurous and slightly mysterious."

## Template — three-act story

```text
<role>You are a story doctor structuring a draft.</role>

<inputs>
<premise>{{1–2 sentences: protagonist + want + obstacle}}</premise>
<protagonist>{{name + flaw + desire + need (often the inverse of desire)}}</protagonist>
<antagonist_force>{{external person, internal conflict, or environment}}</antagonist_force>
<theme>{{the question the story argues about}}</theme>
<genre>{{convention set this needs to satisfy}}</genre>
<length>{{short / novella / novel}}</length>
</inputs>

<output>
## Act I — Setup (≈25%)
- Opening image: ...
- Inciting incident: ...
- Lock-in (point of no return): ...

## Act II — Confrontation (≈50%)
- Rising action: 3–4 escalating beats
- Midpoint: ... (false victory or false defeat)
- All Is Lost: ...

## Act III — Resolution (≈25%)
- Climax: how the protagonist's CHANGE produces the outcome
- Final image: ... (rhymes with opening image)

## Theme statement
The argument the structure makes about {{theme question}}.
</output>
```

The "rhyme" between opening and final image is the highest-leverage technique most drafts miss.

## Template — character design

```text
<role>You design characters who feel inevitable, not arbitrary.</role>

<output>
## Surface
- Name, age, role, era.
- Visual silhouette (one sentence).
- Distinctive feature (something a sketch artist could caricature).

## Interior
- Want (what they CHASE).
- Need (what they LACK and don't know they lack).
- Wound (what hurt them and shapes the want).
- Lie they believe.
- Truth they will (or won't) accept.

## In the world
- Job and how it pays the bills.
- Closest relationship and what's broken in it.
- Status: rising / falling / steady.
- What they say vs. what they do (one example each).

## Voice
- Vocabulary level and register.
- Recurring rhythm or tic.
- One thing they NEVER say.

## Useful contradictions
Two traits that pull against each other.
</output>
```

## Template — dialogue with distinct voices

```text
<role>You write dialogue where each character is recognizable from one line.</role>

<inputs>
<scene_purpose>{{what changes between the beginning and end of the scene}}</scene_purpose>
<characters>
- {{name}} — voice card from the character template above
- {{name}} — voice card
</characters>
<setting>{{where + sensory anchor + a constraint that pressures the conversation}}</setting>
<subtext>{{what each character WANTS but won't say plainly}}</subtext>
</inputs>

<constraints>
- Each character's first line should be recognizably theirs.
- ≤ 2 lines of action / description between speakers.
- No "as you know, Bob" exposition.
- The scene ends when {{purpose}} resolves or the resolution is denied.
</constraints>

<output>
Scene formatted as:
{{NAME}}: line.
*action beat or sensory detail*
{{NAME}}: line.
</output>
```

## Template — worldbuilding (focused, not exhaustive)

Avoid the "tell me everything about your fantasy world" trap — produce only what the story needs.

```text
<role>You build worlds in service of story, not as ends in themselves.</role>

<task>Develop the world details required for {{specific scene or arc}}.</task>

<output>
For each axis, output ONLY what affects the upcoming scenes:

## Geography
What location and what does it physically afford or deny?

## History
ONE event that shaped the present situation.

## Power & economy
Who has resources, who lacks them, what's currently traded?

## Culture
ONE custom that creates pressure or possibility for the protagonist.

## Magic / technology / unique element
ONE rule, including its cost.

## Open contradictions
Things you've decided NOT to decide yet, and what evidence you'll wait for.
</output>
```

The cost rule for magic / technology is the single most important worldbuilding constraint — magic without cost is conflict-killing.

## Template — song structure

```text
<role>You write songs in {{genre}}, focused on emotional arc.</role>

<inputs>
<concept>{{one sentence — what is this song about emotionally}}</concept>
<form>{{verse-chorus-verse-chorus-bridge-chorus | AABA | other}}</form>
<key_and_tempo>{{e.g. C major, 96 BPM, 4/4}}</key_and_tempo>
<reference_tracks>{{1–2 songs whose vibe you want to echo}}</reference_tracks>
</inputs>

<output>
## Title

## Verse 1 (4 lines)
...

## Pre-chorus (2 lines, building)
...

## Chorus (4 lines, hook on the first line)
...

## Verse 2 ...

## Bridge (different perspective or zoom)
...

## Final chorus
(modified to reflect change)

## Notes
- Chord progression sketch.
- Where the production should grow / drop / strip back.
</output>
```

## Template — sound design

```text
<role>You design layered soundscapes for {{film | game | podcast | ambient track}}.</role>

<output>
## Layers (foreground → background)
1. Accents:    occasional, attention-grabbing — ...
2. Foreground: action sounds tied to events — ...
3. Mid-ground: continuous textures — ...
4. Foundation: the ambient bed — ...

## Music score
Tone, instrumentation, motif, dynamic curve over the scene.

## Silence
When and why.
</output>
```

## Template — game design (mechanic / level / character)

```text
## Mechanic
- What does the player DO (verb)?
- What feedback do they get (visual + audio + haptic)?
- What's the failure mode?
- How does mastery feel different from competence?

## Level
Layout sketch (ASCII), pacing curve (calm → tense → climax → release),
secrets, environmental storytelling, the ONE thing this level teaches.

## Character (NPC or playable)
Visual silhouette, abilities (≤3 in plain English), behavior pattern,
weakness, lore one-liner.
```

## Template — 10 / 5 / 3 / 1 brainstorm

```text
<task>Generate ideas for {{problem}}.</task>

<output>
## 10 conventional ideas
Things competent people would propose.

## 5 unusual angles
Reframes — change the audience, the medium, the scale, the constraint.

## 3 boundary-pushing concepts
Things that violate a default assumption.

## 1 synthesis
A single idea that combines the strongest elements of the above.
</output>
```

A reliable structure for ideation that avoids the "give me 10 ideas" trap (where you get 10 mediocre ones).

## Anti-patterns

- **"Make it creative."** Average art.
- **One-slot image prompts** ("a logo for my company"). Fill all six.
- **Worldbuilding without story-need.** Bottomless lore that delivers no scenes.
- **Dialogue without subtext.** Reads like a transcript, not a scene.
- **Magic without cost.** Removes conflict; story collapses.
- **Mood with too many adjectives.** Three is one too many.
- **Generic style references.** "Cool" / "modern" / "epic" — name actual artists, films, eras.
- **Skipping negative prompts on image gen.** Watermarks, distorted hands, extra fingers.

## Cost & routing

| Task | Tier |
|---|---|
| Image-gen prompt writing | cheap to mid |
| Story-structure pass | mid |
| Polished prose generation | mid → frontier |
| Brainstorming (n=10–20 variants) | mid with high temperature |
| Brand-aligned creative on tight voice | frontier with cached brand-voice few-shot |

## Cross-references

- `jimmy-skills@prompt-engineering-multimodal` — full image-gen + audio/video patterns.
- `jimmy-skills@prompt-engineering-role` — character / writer / designer roles.
- `jimmy-skills@prompt-engineering-few-shot` — voice & style transfer via examples.
- `jimmy-skills@prompt-engineering-chain` — outline → scenes → polish.
