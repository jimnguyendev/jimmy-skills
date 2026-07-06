---
name: outcome-thinking
description: Use this skill when the user needs to distinguish outcomes from outputs and impacts, define behavioral outcomes, choose leading indicators, or write outcome-based OKRs. Apply it for management, product, or tech lead questions about value, user behavior, or measuring success, even if the user does not explicitly say "outcomes." For roadmap conversion and journey maps, prefer `outcomes-based-planning`. For org structure, backlog trust, and cross-functional operating model, prefer `organizing-for-outcomes`. For internal change programs and adoption experiments, prefer `outcomes-driven-transformation`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Outcome Thinking

Use this skill when the core problem is that a team is managing work by features, tasks, or delivery dates instead of by behavioral change.

## Boundaries

- Use `outcomes-based-planning` when the task is to rewrite a roadmap, use a journey map, or turn roadmap items into hypotheses.
- Use `organizing-for-outcomes` when the problem is team structure, backlog trust, cross-functional workflow, or defining "done" for outcome work.
- Use `outcomes-driven-transformation` when the target of change is internal stakeholder behavior or organizational adoption.

## Default procedure

1. Separate the discussion into:
   - impact: business result
   - outcome: human behavior change
   - output: thing the team builds or does
2. Rewrite vague goals into observable user, customer, or employee behaviors.
3. Choose leading indicators that predict the business result instead of only lagging indicators that report it after the fact.
4. Express uncertain beliefs as hypotheses.
5. Hand off to planning or transformation workflows when the next step is roadmap design or organizational change.

## Defaults

- Treat outcomes as changes in human behavior, not as feature delivery.
- Treat outputs as possible means, not as goals.
- Use impact to explain why the work matters, but do not give a delivery team only impact and call it a plan.
- Prefer behavioral key results in OKRs over activity or feature key results.
- If the team cannot explain what behavior will change, the proposal is still output-thinking.

## Working heuristics

- "Increase revenue" is usually impact, not an actionable team outcome.
- "Users log in 4+ times per week" is a candidate outcome if it is tied to the business result.
- Leading indicators should help the team decide what to do next, not just report historical performance.
- MVP language belongs downstream in planning or transformation work; this skill's job is to define the behavior and measure, not the rollout plan.

## Validating a leading indicator

A leading indicator is a hypothesis until evidence says otherwise. Grade it on an evidence ladder, cheapest first:

1. **Cohort correlation** — do users who exhibit the behavior show meaningfully better lagging results (retention, conversion, goal achievement) than users who do not? No separation means the indicator is a superstition.
2. **Relative-time impact chart** — align users at the first time they performed the behavior and compare the lagging metric before vs after. Stronger evidence, still observational (self-selection applies).
3. **Randomized experiment** — the only causal proof. Hand off to `jimmy-skills@experimentation-analytics`.

Pair every outcome metric with a **counter-metric**: the behavior you push can be gamed, and the counter-metric catches the damage (pushing "reflections completed" while skip or rushed-completion rates climb means the team is nagging users, not helping them). An outcome without a counter-metric is not ready for an OKR. For structuring the metric set itself, use `jimmy-skills@product-metrics-design`.

## Gotchas

- Teams often relabel current feature work as OKRs without changing how decisions are made.
- A completed feature can be correct by spec and still create no value.
- Outcome language without measurement is still vague planning.
- An unvalidated leading indicator quietly becomes a vanity metric with better branding.
- Impact targets that are far outside team control create confusion rather than accountability.

## References

- Read [references/outcome-thinking.md](references/outcome-thinking.md) when the task involves defining outcomes, writing better OKRs, choosing metrics, or turning assumptions into experiments.
