# Metrics Design Patterns — reference

> Synthesized from: datachallenges.top (Break-even Volume, Lagging & Leading Metrics, Free-to-Paid Conversion series), GrowthBook "What is A/B testing" guide, Amplitude chart docs, and the Playtime 2019 talks (Duolingo, Dailymotion, Monzo). Official docs of the tools remain authoritative.

## 1. Metric hierarchy

| Tier | Role | Cadence | Example (test-prep) |
|---|---|---|---|
| North Star — leading | Tests the core behavioral bet | Weekly | % learners returning to practice within 7 days of first review touch |
| North Star — lagging | Proves real-world impact | Quarterly (holdout) | Verified band/mastery delta |
| Initiative primary | Ship/kill criterion, 1 per initiative | Per experiment (~2 weeks) | Reflect completion rate |
| Secondary | Understand side effects | Per experiment | D7 return, tests/user |
| Guardrail / counter-metric | Must-not-harm | Continuous | Skip rate, abandon rate, first-conversion churn, cost ceiling |

Rules: exactly one primary per initiative (Dailymotion's lesson: "we set too many KPIs to improve at once"); guardrails are chosen before launch, never after results arrive (that is metric shopping).

## 2. Leading vs lagging

- Leading: behavioral, high-frequency, readable within the experiment window. Use for decisions.
- Lagging: business results (revenue, LTV, exam outcomes). Use for release monitoring and quarterly holdouts.
- Validate that a leading metric actually leads: cohort correlation first, relative-time impact chart second, randomized experiment as the gold standard. A leading metric that never predicted the lagging one is a superstition.

## 3. Denominator patterns

Most metric disputes are denominator disputes. Declare one of:

- All registered users (acquisition framing)
- Active users in window (engagement framing)
- Users exposed to the surface (experiment framing — required for A/B analysis)
- Users who performed the prerequisite event (depth framing — e.g., "avg days performed" divides by users who did the event at least once, not by all MAU)

Watch for >100% artifacts when numerator events fire for users outside the denominator definition.

## 4. Counter-metric catalogue

| If you push... | Watch... |
|---|---|
| Conversion to paid | First-conversion churn / refunds (29% post-payment cancel = alarm in the source case) |
| Faster funnel completion | Downstream quality (exam pass rate, error rate) |
| Notification engagement | Unsubscribe / notification disable rate |
| Session count | Completion per session (dồn cục vs habit) |
| Prompted actions (e.g., Self Reflect) | Skip / dismiss / rushed completions |
| Marginal-cost feature usage | Cost per unit vs ceiling |

## 5. Unit-economics worksheet (from Break-even Volume)

1. List per-unit variable cost for each product mix component (e.g., Writing review vs Speaking review — Speaking costs more).
2. Weighted cost = sum(mix% x unit cost). Recompute when the mix shifts; run sensitivity on the worst plausible mix.
3. Ceiling: weighted cost x projected volume <= infra budget. This is a standing guardrail, not a one-off analysis.
4. Report the same number three ways: monthly (leadership), weekly (squad), daily (ops) — "one number, three languages."

## 6. Anti-vanity notes

- A metric that can only go up (cumulative counts) is a chart, not a metric.
- Averages hide bimodality; check the distribution before trusting a mean (see stickiness distributions in `retention-engagement-analysis`).
- Cap count metrics at p99 for experiment comparisons.
- For deadline-driven products, "churn after goal achieved" is graduation — segment cohorts by goal date where possible.
