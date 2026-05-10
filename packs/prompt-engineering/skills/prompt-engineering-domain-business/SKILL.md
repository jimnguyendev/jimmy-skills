---
name: prompt-engineering-domain-business
description: "Templates for business communication and operations: emails by recipient/purpose/tone, meeting agendas, OKRs, decision frameworks (SWOT, weighted criteria), strategic plans, and SOPs. Each template forces the user to specify the context that AI cannot infer."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Pairs with retrieved context (CRM, calendar, internal docs) for grounded answers."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Domain — Business & Productivity

The recurring rule: **AI structures thinking; the human supplies real-world context.** Every template here forces the operator to provide the inputs the model cannot guess (recipient, stakes, constraints, deadlines).

## Template — email

```text
<role>You are a professional drafting an email on the sender's behalf.</role>

<inputs>
<recipient>{{name + role + relationship to sender (peer, boss, client, vendor)}}</recipient>
<purpose>{{ONE outcome: get approval, share update, request info, decline, etc.}}</purpose>
<key_points>
1. ...
2. ...
3. ...
</key_points>
<tone>{{2–3 adjectives, e.g. "warm but direct"}}</tone>
<sender>{{name + role}}</sender>
<context>{{prior thread summary, deadline, stakes — only what the recipient needs}}</context>
</inputs>

<constraints>
- Subject line ≤ 7 words; states the purpose, not the topic.
- Open with the ask or update in the first sentence.
- ≤ 150 words unless complexity requires more.
- One clear CTA (or "FYI, no action needed").
- No filler ("hope this finds you well").
</constraints>

<output>
Subject: ...
Body: ...
</output>
```

The single highest-leverage slot is `purpose`. "Email about the project" produces wallpaper. "Email to get approval to delay shipping by 2 weeks" produces a useful draft.

## Template — decision framework

```text
<role>You are a strategy advisor structuring a decision.</role>

<task>Apply {{SWOT | weighted criteria | competitive positioning | pre-mortem}} to this decision.</task>

<inputs>
<decision>{{phrased as a binary or multi-option choice}}</decision>
<options>{{2–5 specific options}}</options>
<criteria>{{what success looks like, with weights if used}}</criteria>
<known_facts>{{what we know — keep tight}}</known_facts>
<unknowns>{{what we don't know yet — equally important}}</unknowns>
</inputs>

<output>
{{framework-shaped output}}
+ One-line recommendation
+ Three risks the recommendation depends on being false
+ One thing to find out before committing
</output>
```

**SWOT** for situational awareness, **weighted criteria** for option selection, **pre-mortem** for risk surfacing. Pick the framework that matches the decision shape.

## Template — meeting agenda

```text
<role>You design meetings that produce decisions, not status updates.</role>

<inputs>
<purpose>{{one sentence: decide X | align on Y | unblock Z}}</purpose>
<duration>{{minutes}}</duration>
<attendees>{{names + role + why each is needed}}</attendees>
<pre_reads>{{links — anything attendees should read before}}</pre_reads>
<desired_outcomes>
1. {{measurable: a decision, an artifact, an alignment}}
</desired_outcomes>
</inputs>

<output>
## Meeting: {{title}}
**Purpose:** ...
**Outcome target:** ...

| Time | Topic | Lead | Outcome |
| --- | --- | --- | --- |
| 0–5  | Context recap | ... | Shared starting point |
| 5–25 | ... | ... | Decision on X |
| 25–35 | ... | ... | Owner + date |
| 35–45 | Wrap | ... | Action items captured |

## Pre-reads
...

## Decisions to make in the meeting
1. ...

## Action item template
- [ ] {{action}} — owner — by date
</output>
```

If a meeting has no `decisions to make`, it is a status update — turn it into an async post.

## Template — OKR

```text
<role>You write outcome-focused OKRs.</role>

<task>Draft Objectives and Key Results for {{team}}, {{quarter}}.</task>

<inputs>
<mission>{{team mission, 1 sentence}}</mission>
<context>{{strategic priorities, constraints, prior-quarter results}}</context>
<themes>{{2–4 themes the team will focus on}}</themes>
</inputs>

<constraints>
- Objectives are qualitative, ambitious, and time-boxed.
- Each Objective has 2–4 Key Results.
- Each Key Result is QUANTITATIVE: a number with a unit and a baseline.
- Avoid activities ("ship feature X") — KRs measure outcomes ("reduce churn from N% to M%").
- Each KR has an owner.
- 70% confidence = appropriately ambitious; 100% means too easy.
</constraints>

<output>
## Objective 1: {{qualitative aspiration}}
- KR1.1 — {{metric}}: {{baseline}} → {{target}} (owner: ...)
- KR1.2 — ...
- KR1.3 — ...

## Objective 2: ...
...

## Risks to OKR achievement
1. ...
</output>
```

## Template — strategic plan

```text
<role>You produce concise plans that connect goals → bets → milestones.</role>

<inputs>
<horizon>{{quarter | half | year | multi-year}}</horizon>
<goal>{{the outcome that defines success}}</goal>
<constraints>{{budget, team size, deadline, dependencies}}</constraints>
<assumptions>{{stated explicitly so they can be challenged}}</assumptions>
</inputs>

<output>
## Goal
One sentence.

## Bets
3–5 named initiatives. Each:
- Hypothesis: "If we X, then Y, because Z."
- Why this and not alternatives.
- Cost / time / risk.

## Milestones
| Phase | Date | Deliverable | Decision point |

## Risks
- {{risk}} — likelihood × impact — mitigation.

## Assumptions to validate
1. ...

## Stop conditions
- We will pause / pivot if: ...
</output>
```

## Template — SOP / runbook

```text
<role>You write SOPs that a new hire can follow without asking questions.</role>

<inputs>
<process>{{name}}</process>
<purpose>{{why this exists}}</purpose>
<inputs_needed>{{what the operator must have before starting}}</inputs_needed>
<expected_duration>{{minutes/hours}}</expected_duration>
<systems>{{tools and access required}}</systems>
</inputs>

<output>
## Procedure: {{name}}
**When to use this:** ...
**You'll need:** ...
**Estimated time:** ...

### Steps
1. {{action}} — expected result. *If you see X instead, see Exceptions.*
2. ...
N. Confirm completion: {{visible signal of success}}

### Exceptions
- **{{symptom}}** → {{action}}; if still failing, escalate to {{owner}}.
- ...

### Decision points
- After step {{N}}: if {{condition}}, branch to {{alt path}}.

### Logging / audit
What to record, where, retention.

### Owner / review cadence
Owner: ... ; review: every {{N}} months.
</output>
```

The exceptions section is what separates a runbook from a wish list.

## Anti-patterns

- **"Write an email about the project."** No recipient, no purpose, no key points → wallpaper.
- **OKRs with activities as KRs.** "Ship feature X" is an activity; KRs measure outcomes.
- **Meetings without decisions to make.** Convert to async update.
- **Decision frameworks without unknowns.** Listing only what you know means the framework just confirms your prior.
- **SOPs without exceptions.** Real operators hit edge cases on day one.
- **Strategic plans without stop conditions.** Plans that can't be killed don't get pivoted.

## Cost & routing

| Task | Tier |
|---|---|
| Routine emails (templated) | cheap |
| Meeting agenda / SOP | cheap to mid |
| OKR / strategic plan | mid → frontier (high stakes, low frequency) |
| Decision framework on novel choice | frontier with reasoning |
| Bulk customer / vendor outreach | cheap with cached few-shot of brand voice |

## Cross-references

- `jimmy-skills@prompt-engineering-role` — different roles for different docs.
- `jimmy-skills@prompt-engineering-output-structured` — table-heavy outputs.
- `jimmy-skills@prompt-engineering-chain` — research → frame → recommend pipelines.
- `jimmy-skills@prompt-engineering-context` — ground in CRM / calendar / internal docs via RAG.
