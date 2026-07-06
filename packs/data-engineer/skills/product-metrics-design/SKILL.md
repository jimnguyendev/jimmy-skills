---
name: product-metrics-design
description: Use this skill when the user needs to choose, define, or defend product metrics — North Star selection, leading vs lagging indicators, primary/secondary/guardrail metrics for a feature or experiment, counter-metrics, denominators, or unit-economics ceilings. Apply it for "what should we measure," "is this a vanity metric," "define success for this feature," or dashboard KPI reviews, even if the user does not explicitly say "metrics design." For running the actual analysis (retention curves, engagement matrix, churn states), prefer `jimmy-skills@retention-engagement-analysis`. For designing the experiment that moves a metric, prefer `jimmy-skills@experimentation-analytics`. For building the pipeline and marts that serve metrics, prefer `jimmy-skills@data-engineering`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Product Metrics Design

Use this skill when the core problem is deciding *what to measure and why* — before any dashboard, mart, or experiment is built. A metric set is a product decision, not a reporting task.

## Boundaries

- Use `jimmy-skills@retention-engagement-analysis` when the metrics are chosen and the task is to compute or interpret retention, engagement, or churn behavior.
- Use `jimmy-skills@experimentation-analytics` when the task is to design a test that moves a chosen metric.
- Use `jimmy-skills@outcome-thinking` when the discussion is still at the outcome-vs-output level and no measurable behavior has been named yet.

## The five filters

Every metric must pass all five before it earns a place on a dashboard:

1. **Behavior, not shipping.** It measures what users *do*, not what the team delivered. "Feature launched" is not a metric.
2. **Leading or lagging — declared explicitly.** Leading metrics (behavior, readable in 1–2 weeks) drive decisions; lagging metrics (revenue, exam-score delta, LTV) prove impact months later. Never use a lagging metric as the ship/kill criterion of a two-week test.
3. **Actionable.** Someone can name the experiment or the state-transition arrow this metric points at. If nobody can act on it, it is decoration.
4. **Paired with a counter-metric.** Every metric you push can be gamed; the guardrail catches the damage. No counter-metric = not ready.
5. **Fits the business cycle.** A test-prep learner churning after passing their exam is a *graduation*, not a failure. Match windows (weekly, not daily) and cohorts (by exam date) to how the product is actually used.

## Default procedure

1. Ask what decision the metric will drive. No decision → stop.
2. Write the North Star pair: one leading (behavior) + one lagging (business/impact proof). Keep them distinct.
3. For each initiative, define exactly one primary metric, a few secondary, and explicit guardrails.
4. Define the denominator before the numerator. Most metric disputes are denominator disputes (all users? active users? users who saw the feature?).
5. Add the unit-economics ceiling if the feature has marginal cost: cost per unit x volume must stay under budget; recompute the weighted cost when the mix shifts.
6. Translate one number into three languages: monthly for leadership, weekly for the squad, daily for operations. Same metric, three framings.

## Worked example (test-prep product)

- North Star leading: % of learners who return to practice within 7 days after first touching the essay-review experience — it tests the core behavioral bet.
- North Star lagging: verified mastery/band delta measured on later submissions (read quarterly via holdout, never via a 2-week test).
- Feature "Self Reflect" primary: reflect completion rate. Guardrail: skip/dismiss rate — if skips spike, the feature is nagging, not helping. Ship decision blocked on the guardrail, not just the primary.
- Monetization primary: free-to-paid conversion. Mandatory guardrail: **first-conversion churn** — in the source case study, 29% of users cancelling right after their first payment was the alarm that conversion had been pushed too hard.

## Anti-vanity checklist

- DAU/streak as a headline KPI for deadline-driven products invites hollow engagement mechanics.
- Time-in-app is two-faced: engaged or lost? Only read it next to a completion metric.
- Counts without denominators ("10k sessions!") are press releases, not metrics.
- A capped mean (p99) protects count metrics from one power user distorting a comparison.

## Trap from a real system

An edtech team shortened a mandatory tutorial and celebrated +20% time-to-first-lesson. The guardrail they almost didn't have showed a 7% drop in final exam pass rate. The metric set caught what the primary alone would have shipped. Design the guardrail *before* you need it.

## References

- Read [references/metrics-design-patterns.md](references/metrics-design-patterns.md) for the metric hierarchy, denominator patterns, unit-economics worksheet, and the full worked example.
