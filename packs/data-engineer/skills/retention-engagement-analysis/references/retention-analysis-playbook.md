# Retention & Engagement Analysis — playbook

> Synthesized from: datachallenges.top (User Activation Moments #01–#04, Churn Analysis Model, DAU Model, Cohort vs Rolling Retention), Amplitude chart docs (Engagement Matrix, Stickiness, Impact Analysis, Personas), and a reverse-engineered Duolingo web dump. Official docs of the tools remain authoritative.

## 1. Lifecycle state machine (weekly)

States and transitions (adapt grace periods to your cycle):

- NEW — first active week. Split SINGLE (1 active day) vs MULTI (2+ days): NEW MULTI typically shows far higher LTV, so week one is the highest-leverage window.
- ACTIVE NEW — active again the following week.
- ACTIVE EVERGREEN — active 3+ consecutive weeks.
- PAUSED (NO ORDER) — inactive >= 1 week from any active state.
- REACTIVE — returned after a pause.
- CHURN — two flows: (a) NEW who stays silent past the grace period (~2 weeks); (b) REACTIVE who goes silent the very next week — a failed winback is the strongest churn signal.

Three companion analyses:

1. **Transition matrix** — P(A → B) per week; compare across cohorts. An experiment claims one arrow (e.g., PAUSED → REACTIVE) and is judged on it.
2. **Reactivation quality** — % of REACTIVE churning next week. Winback that pulls users back without keeping them is a real failure hiding inside a positive-looking count.
3. **Leading churn signals** — compare each user's last week to their own 3–4 week baseline (submits, active days, session length). Flag at-risk before they pause; nudges are cheapest at this stage.

Deadline-driven products: segment by goal/exam date when available; post-goal CHURN is graduation, not failure.

## 2. Engagement matrix

- X = % of period-active users who performed the event (breadth). Average over ~3 periods to smooth.
- Y = avg **days** performed per period among users who performed it (depth). Denominator = performers, not all actives.
- Offer avg **times** as an alternate Y: times measures volume, days measures return frequency. Events that rank high on times but low on days are binge patterns.
- Quadrant lines at the **median** of plotted events (self-adapting, always populates four quadrants). Log scale when volumes span orders of magnitude.
- Third layer — **entropy** of activity spread across sub-periods (e.g., 4 weeks): H = -sum(p_i log2 p_i). High H = evenly spread (habit); H near 0 = concentrated burst (churn risk). Two users with identical "4 active days" can differ completely here.

Quadrant actions: core (broad+deep) = protect, experiment here; deep-narrow = discovery problem, surface it; broad-shallow = stickiness problem or accept as transit; narrow-shallow = question its existence.

## 3. Stickiness distribution

Histogram of days-per-period per user for one event. Uses:

- Find the "break" between casual and regular users → set the Habit-moment threshold empirically instead of by decree.
- Detect bimodality that averages hide.
- Validate: retention of users above the threshold should clearly beat those below; otherwise the threshold is arbitrary.

## 4. Activation ladder (Setup → Aha → Habit)

1. **Setup moment** — completed the golden path to first value (e.g., first test started). Analyze the path stepwise for the biggest drop.
2. **Aha moment** — the behavior combo in the first N days that best separates retained from churned users. Method: behavioral cohorts — compare retention of users with vs without each candidate combo; pick the strongest separator. Never declare an aha moment without this comparison.
3. **Habit moment** — repetition pattern (e.g., >=3 events on >=3 distinct days in 14 days), threshold from the stickiness break.

Companion metrics: activation velocity (time to each rung; survival analysis shows the "golden window" for intervention) and quality activation rate (reached Aha AND still active at day N — guards against hollow activation).

## 5. Impact analysis (before/after around first behavior)

- Align every user at T0 = first occurrence of the treatment behavior; plot the outcome rate for ±N relative weeks.
- Metrics: average (among users who did the outcome), active % (among users active in the interval), frequency distribution.
- **Caveats (mandatory):** self-selection — users who adopted the behavior may already differ; check alternate behaviors occurring around the same T0 by charting them too; small N makes curves noisy.
- Role in the workflow: grade evidence strength for prioritization (e.g., the confidence input of an ICE score). A strong impact chart earns an experiment slot; it never replaces the experiment.

## 6. Behavioral clustering (personas)

Quarterly k-means over behavioral features (events per week, active-day spread, feature mix, time-of-day). Expect archetypes like crammer / daily grinder / reviewer / browser in learning products. Uses: targeting attribute for experiments; mandatory segment cut when reading results (an overall winner can lose inside a key persona). Cluster count is exploratory — try several; for new users cluster on first-day behavior only.

## 7. Analysis hygiene

- Exposure, not assignment, defines analysis populations (Duolingo `treated` vs `destiny`).
- State the retention definition (cohort vs rolling) on every chart.
- Distributions before averages; medians for boundaries; caps for outliers.
