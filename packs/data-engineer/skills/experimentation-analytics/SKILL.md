---
name: experimentation-analytics
description: Use this skill when the user needs to design, run, or read an A/B test or experimentation program — hypotheses, one-pagers, sample size and power, primary/guardrail metrics, exposure vs assignment, SRM, sticky bucketing, sequential testing, choosing between A/B, A/B/n, multivariate, bandit, holdout, cluster, or CUPED, and deciding when NOT to test. Apply it for "how do we A/B test this," "is this result significant," "the test looks flat," or experiment platform reviews (GrowthBook, Amplitude, Statsig), even if the user does not explicitly say "experiment." For picking the metrics themselves, prefer `jimmy-skills@product-metrics-design`. For observational analyses that generate hypotheses, prefer `jimmy-skills@retention-engagement-analysis`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Experimentation Analytics

Use this skill when the team wants causal answers. Randomized experiments are the only tool here that proves causality; everything else is evidence for prioritization.

## Boundaries

- Use `jimmy-skills@product-metrics-design` when primary/guardrail metrics are not yet defined — an experiment without a declared metric set is not ready.
- Use `jimmy-skills@retention-engagement-analysis` for cohort/impact analyses that feed hypotheses into the experiment backlog.
- Use `jimmy-skills@outcome-thinking` when the team has not yet named the behavior it wants to change.

## Non-negotiables

1. **One-pager before code.** Hypothesis ("[specific change] will cause [measurable effect] because [research-based reason]"), one primary metric, guardrails, success threshold, owner, read date, and existing evidence (an impact chart grades the Confidence in ICE).
2. **Power analysis before launch.** Four inputs: baseline, minimum effect worth shipping, confidence (95%), power (80%). Divide required units by daily eligible traffic to get duration; run at least two full business cycles.
3. **Assign ≠ Expose.** Only users who actually saw the variant count. Fire exposure at the render point, not at flag resolution or prefetch. Route-level flags may resolve in a loader (dedup makes it safe); item-level experiments must evaluate at the component that renders them.
4. **After launch, leave it alone.** Monitor only bugs, guardrail collapses, and SRM (sample ratio mismatch = broken assignment = invalid results). No peeking-based early stops unless sequential testing is enabled.
5. **A/A test before trusting any pipeline**, per client platform. Expect 1 in 20 A/A tests to look "significant" at 95%.

## Choosing the design

| Problem shape | Design | Note |
|---|---|---|
| Two ways to build one thing, multiple metrics matter | Standard A/B 50/50 | Default choice |
| 3+ distinct hypotheses at once | A/B/n | Sample scales with variant count |
| Repeated decision, one objective, 5+ variants (notification copy, model/prompt pick) | Multi-armed bandit | Optimizes; does not explain. Poor fit for learning why |
| Long-term or cumulative effect (does the feature improve real outcomes?) | Holdout (~5% withheld for a quarter) | The only honest way to read slow outcomes like learning gains |
| Users influence each other (classrooms, teams) | Cluster randomization at group level | Never split a classroom between variants |
| Metric has per-user history | Add CUPED | Pre-experiment data cuts required sample (~+20% traffic equivalent) |
| Tail behavior matters (latency, slowest users) | Quantile test (p95/p99) | Means hide tail damage |
| Obviously-right change (bug fix, accessibility) | No test — ship behind a flag; optionally non-inferiority check | Testing it wastes a slot and can be unethical |

Also skip experiments when: no product-market fit yet (interview instead), sample too small for the effect size (make bolder changes), brand identity changes, or no internal agreement on what success means.

## Capacity is a physical constraint

Concurrent experiments compete for eligible users and must be mutually exclusive on shared surfaces (use platform namespaces). Compute capacity = eligible traffic / required sample per test — and put that number in the team ritual. The true cost of a test = build + 2–3 weeks of waiting + reading + **the test you could not run instead**. Small teams: start with a max of ~3 concurrent.

## Reading results

- Wait for full sample plus the metric window (a 7-day metric needs 7 more days after the last exposure).
- Check SRM, then guardrails, then segments (platform, plan, persona) — an overall winner can lose in a key segment.
- Statistical vs practical significance: a "significant" 0.3% lift may not pay for its maintenance; an 85%-probable winner with clean guardrails and low cost is often shippable. Thresholds are set before launch and never moved after.
- Inconclusive means one of three things: underpowered, effect below the minimum worth shipping, or genuinely no difference. Losses are documented learnings — record them so the same test is not rerun in six months.

## Worked example (test-prep product)

Foundation vs experiment (Dailymotion pattern): a new essay-review experience is a *foundation* — release it through staged tests: (1) backlash check on 10–20%, reading guardrails only; (2) stabilize; (3) only then A/B the variants (inline vs sidebar annotations) with one primary each; (4) ship and keep monitoring. Reversed order — A/B-ing polish before the foundation is stable — produces weeks of unreadable results.

## Trap from a real system

Monzo ran high-tempo tests on one funnel, ran out of eligible users, and **shared a control group between two experiments** to squeeze one more in. A bug assigned one experiment's treatment to the other's control: both experiments and the next sprint's tests were invalidated — weeks lost. Never work around the platform's exclusivity model; if there is no capacity, there is no test.

## References

- Read [references/experimentation-playbook.md](references/experimentation-playbook.md) for the one-pager template, power-analysis walkthrough, exposure rules, statistics notes (Bayesian vs frequentist, sequential), and the program-maturity ladder.
