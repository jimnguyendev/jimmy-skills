---
name: prompt-engineering-role
description: "Activate expert behavior in a single message by assigning a specific role or persona. Covers the basic Expert / Professional / Teacher patterns, advanced Compound / Situational / Perspective constructions, the Role Stack technique for layering identity + audience + style, and anti-patterns. Use when one message needs a focused expert lens — for persistent agent identity, use prompt-engineering-system-prompt instead."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Works with any chat or completion model."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep AskUserQuestion
---

# Role-Based Prompting

Assigning a role reshapes what the model considers "likely" to say next. Saying *"You are a senior penetration tester"* activates security vocabulary, threat-modeling reasoning, and adversarial framing that a generic assistant averages away. The technical mechanism is simple: the role conditions the model's probability distribution toward the patterns it saw from that kind of professional in training.

This skill is for **single-message** role focusing. If you need the same identity across many turns, build a system prompt with `prompt-engineering-system-prompt` instead.

## When to use

- A user message needs an expert lens (security, legal, medical, devops, finance, pedagogy, etc.).
- You want the model to filter knowledge through a specific viewpoint.
- You want consistent vocabulary, detail level, and reasoning style for one task.
- Quality is the goal — roles are not a cost optimization, they spend tokens.

## Three Basic Patterns

### Expert pattern

Convey deep, authoritative knowledge by specifying **field + experience duration + specialty focus**.

```text
You are a senior security engineer with 12 years of experience focused
on cloud-native authentication and identity. Review the auth flow below
for vulnerabilities…
```

### Professional pattern

Ground the role in workplace context — **job title + organization type** — to add institutional norms.

```text
You are a Staff SRE at a high-traffic e-commerce company. We just got paged
for elevated 5xx on checkout. Walk through the on-call response…
```

### Teacher pattern

Match explanation complexity to audience expertise.

```text
You are a CS professor explaining concurrency to a third-year undergraduate
who has written threaded code in Java but never used channels. Explain
Go's CSP model…
```

The teacher pattern is the only one where audience belongs in the role itself; for the other two, audience goes in `<context>`.

## Three Advanced Constructions

### Compound roles

Merge two identities to blend perspectives.

```text
You are both a clinical pharmacist and a software engineer. Review this
medication-reminder app's data model for both medical correctness and
implementation soundness…
```

Use compound roles when two domains genuinely overlap on the task. Don't stack roles that don't share a problem space — you get an average, not a synthesis.

### Situational roles

Place the role in a specific scenario to shape **content and communication tone**.

```text
You are a senior architect leading a design review where the team has
already shipped half the feature. Your job is to surface risks without
demoralizing the team or demanding a rewrite…
```

Situational framing is how you get the right *register*, not just the right knowledge.

### Perspective roles

Evaluate a situation from a stakeholder-specific viewpoint to surface their priorities.

```text
You are the head of Compliance reviewing this product launch plan.
List only the items that would block your sign-off and why…
```

Run the same prompt with three different perspective roles (Engineering / Compliance / Sales) to triangulate concerns.

## The Role Stack

Layer **domain expertise + audience awareness + style guidelines** into one identity. Each layer narrows the output further.

```text
You are:
- A senior backend engineer who has shipped Postgres at scale (domain).
- Writing for a junior engineer in their first month on the team (audience).
- Communicating in our team style: direct, evidence-first, no hedging,
  cite EXPLAIN output line numbers when discussing query plans (style).

Review the migration below…
```

The Role Stack is the production-grade pattern. It is also the most common starting point for distilling into a `prompt-engineering-system-prompt` once the role proves itself.

## Combining roles with other techniques

- **Role + Few-Shot.** Examples carry tone and format; the role supplies the why. Use both when the desired output style is unusual or domain-specific.
- **Role + Chain of Thought.** Most expert roles already encourage step-by-step reasoning ("As an architect, walk through the trade-offs…"). Adding an explicit CoT trigger is redundant unless the task requires verifiable steps (math, debugging, planning).
- **Role + Output Contract.** Always specify `<output>` separately. The role determines *content*, not *shape*.

## Domain Catalog

Useful first-draft roles by category:

**Technical**

- Software architect (system design, trade-off analysis)
- Security specialist (threat modeling, vulnerability triage)
- DevOps / Platform engineer (infra-as-code, deployments)
- Database / Performance engineer (query plans, indexing)
- Distributed systems engineer (consistency, partial failure)

**Creative**

- Copywriter (persuasion, conversion)
- Screenwriter (narrative arc, dialogue)
- UX writer (brief, action-oriented interface text)
- Brand strategist (voice, positioning)

**Analytical**

- Business analyst (requirements, stakeholder translation)
- Research scientist (evidence-based, uncertainty acknowledged)
- Financial analyst (risk-adjusted return)
- Product analyst (funnel, retention, cohort)

**Educational**

- Socratic tutor (questions, not answers)
- Instructional designer (learning progression)
- Technical writer (docs, runbooks)

Treat these as starting points; specialize before using.

## Anti-patterns

- **Generic roles.** "You are an expert" gives no activation signal. Add field, depth, focus.
- **Conflicting roles.** "You are an expert pen-tester and an enterprise sales lead." The model averages; both lenses degrade.
- **Unrealistic expertise.** "You are the world's leading authority on everything." T-shaped (deep in one, broad in adjacent) is more believable and produces better outputs.
- **Role as decoration.** A role you don't need spends tokens for nothing. Drop it for simple factual or formatting tasks.
- **Role as safety bypass.** Roles do not override system rules and should not be used to attempt jailbreaks; modern models resist this and your eval set should catch attempts.

## Quick template

```text
You are a <role with depth + focus>.
Your audience is <audience>.
Your style is <2–3 adjectives>.

<task>
<inputs>
<constraints>
<output>
```

That fits inside the Prompt Frame from `prompt-engineering-core` and is the recommended starting point for any new role-based prompt.

## Cross-references

- `jimmy-skills@prompt-engineering-core` — the Prompt Frame this slots into.
- `jimmy-skills@prompt-engineering-system-prompt` — when the role needs to persist across turns.
- `jimmy-skills@prompt-engineering-few-shot` — pair with examples for tone.
- `jimmy-skills@prompt-engineering-pitfalls` — generic-role and conflicting-role checks.
