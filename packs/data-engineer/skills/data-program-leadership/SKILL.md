---
name: data-program-leadership
description: Use this skill when the user needs to lead a data initiative, scope a roadmap, assess data maturity, define roles, avoid delivery pitfalls, or turn business questions into a staged data program. Apply it for tech lead and manager questions about stakeholders, requirements, rollout plans, project risk, or proving data value early, even if the user does not explicitly say "leadership" or "program management."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Program Leadership

Use this skill when the technical question is entangled with ownership, sequencing, stakeholder alignment, or adoption risk.

## Default procedure

1. Determine the current data maturity stage and what the organization can actually operate.
2. Identify the immediate business questions that must be answered first.
3. Run domain discovery for unknown schemas; prioritize domains where schema is known and business urgency is highest.
4. Scope a phase that can show trustworthy results quickly.
5. Design multi-team data strategy (Data Mesh Lite for mid-size teams, pure Data Mesh only at 100+ engineers).
6. Assign clear owners for architecture, business definitions, validation, and sponsorship.
7. Reduce risk by validating numbers with users before broad rollout.

## Defaults

- For stage 1-2 teams, prove value with a small reporting slice before broad platform ambition.
- Show results early instead of waiting for the "perfect" platform.
- Gather enough requirements to start, then iterate with users.
- Prefer cutting scope over cutting quality or validation.
- Model for business questions, not as a copy of source schemas.

## Common pitfalls

There are 14 documented pitfalls that cause data projects to fail (see references for full detail):

- executives underestimating BI complexity ("it's just a dashboard")
- choosing technology for fashion instead of fit
- collecting too many requirements (analysis paralysis) or too few (building blind)
- showing unvalidated numbers to users (trust destruction)
- inexperienced consultants or offshore teams without oversight
- giving full ownership to external parties with no internal visibility
- no knowledge transfer (bus factor of one)
- budget cuts mid-project (cut scope, not quality)
- hard deadlines forcing rushed discovery and skipped validation
- modeling the warehouse as a copy of source schemas instead of business questions
- poor dashboard performance compared to existing tools
- over-designing or under-designing the architecture
- poor communication between technical and business teams
- failing to define metric ownership

## Gotchas

- The first broken dashboard can destroy trust faster than a late dashboard.
- Most failures in data programs come from people and process, not missing tools.
- A tech lead on an early-stage team often has to act as architect, analyst bridge, and delivery owner at the same time.
- If a metric has no business owner, it will drift.

## References

- Read [references/program-leadership.md](references/program-leadership.md) when the task involves maturity assessment, team roles, the 14 pitfalls, 6 delivery tips, multi-team data strategy (Data Mesh Lite), domain discovery, phased rollout planning, or stakeholder alignment.
