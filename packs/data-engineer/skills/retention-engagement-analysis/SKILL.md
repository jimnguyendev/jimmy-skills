---
name: retention-engagement-analysis
description: Use this skill when the user needs to analyze user retention, engagement, activation, or churn behavior — cohort and rolling retention, user lifecycle state machines, transition matrices, engagement matrix (breadth x depth), stickiness distributions, activation moments (setup/aha/habit), or before-after impact analysis around a first behavior. Apply it for "why do users leave," "which features drive retention," "define our aha moment," or "is this feature a habit," even if the user does not explicitly say "retention analysis." For choosing which metrics deserve a dashboard, prefer `jimmy-skills@product-metrics-design`. For proving causality with a randomized test, prefer `jimmy-skills@experimentation-analytics`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Retention & Engagement Analysis

Use this skill when the metrics exist and the task is to *understand behavior*: who stays, what makes them stay, and where users fall off. These analyses generate hypotheses; they do not prove causality — that is the experiment's job.

## Boundaries

- Use `jimmy-skills@product-metrics-design` when the question is which metrics to define in the first place.
- Use `jimmy-skills@experimentation-analytics` when a hypothesis from these analyses needs causal proof.
- Use `jimmy-skills@data-value-patterns` when the blocker is building the sessions/aggregates these analyses need.

## Analysis toolbox — pick by question

| Question | Tool | Key detail people get wrong |
|---|---|---|
| Do users come back? | Cohort retention (fixed-week cohorts) vs rolling retention | The two definitions give different numbers — always state which one you are using |
| What lifecycle state is each user in? | Weekly state machine: NEW (single/multi) → ACTIVE → EVERGREEN → PAUSED → REACTIVE → CHURN | Grace periods are business decisions; a REACTIVE user who goes silent next period churned *harder* than average |
| Which arrow should we move? | Transition matrix P(state A → B per week) | An experiment should target one named arrow, not "retention" in general |
| Which features are truly engaging? | Engagement matrix: %MAU (breadth) x avg days performed (depth), quadrant lines at the **median** of plotted events | Depth denominator = users who did the event, not all MAU; add entropy to separate habit from binges |
| Is usage a habit or a binge? | Stickiness distribution (days-per-period histogram) + entropy of activity spread | Averages hide bimodality; days > times as a habit signal (10 times in 1 day is not a habit) |
| What is our aha moment? | Activation ladder: Setup → Aha → Habit, each validated by correlation with later retention | An unvalidated aha moment is a slogan. Compare retention of users with vs without the candidate behavior combo |
| Did behavior change after X? | Impact analysis: align all users at T0 = first time doing X, compare outcome rate ±N weeks | Self-selection — users who did X may differ. Use as evidence strength for prioritization, never as causal proof |
| Who is about to churn? | Leading churn signals: compare a user's recent activity to their own 3–4 week baseline | Compare to self, not to population averages |

## Default procedure

1. Fix definitions first: active = which event, period = day or week, denominator = who. For deadline-driven products prefer weekly periods and treat post-goal churn as graduation.
2. Build the funnel and find the biggest drop before analyzing anything subtle.
3. Run the engagement matrix to shortlist events worth attention (core quadrant + high entropy).
4. Define and validate the activation ladder against retention data.
5. Build the weekly state machine and transition matrix; name the one or two arrows the team will target.
6. For each candidate intervention, run an impact chart to grade evidence strength, then hand the strongest ones to `jimmy-skills@experimentation-analytics`.

## Worked example (test-prep product)

State machine on weekly `test_submit`: the matrix showed REACTIVE → CHURN at 56%/week — winback campaigns were pulling learners back but not keeping them. The team stopped celebrating "users reactivated" and targeted the REACTIVE → ACTIVE arrow instead, with reactivation *quality* (retained next week) as the primary metric. Meanwhile the engagement matrix showed `leaderboard_view` wide but shallow with low entropy (binge pattern) — deprioritized despite its impressive raw counts.

## Trap from a real system

Duolingo's per-user experiment object carries five fields: `destiny` (precomputed deterministic bucket), `eligible`, `condition`, `treated`, `contexts`. The pair that matters here: **`destiny` (assigned) vs `treated` (actually saw it)**. Any before/after or funnel analysis that counts assigned-but-never-exposed users dilutes the effect and understates every result. Always analyze on exposure, not assignment.

## References

- Read [references/retention-analysis-playbook.md](references/retention-analysis-playbook.md) for state definitions, matrix SQL sketches, engagement matrix and entropy formulas, activation-ladder validation, and impact-chart caveats.
