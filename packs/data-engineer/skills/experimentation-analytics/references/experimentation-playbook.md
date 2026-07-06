# Experimentation — playbook

> Synthesized from: GrowthBook "What is A/B testing — the complete guide" and docs, Playtime 2019 talks (Duolingo — Karin Tsai, Dailymotion — Jean-Loup Yu, Monzo — Bruno Vaz Moco), and a reverse-engineered Duolingo web dump. Official platform docs (GrowthBook, Amplitude, Statsig) remain authoritative for their features.

## 1. One-pager template (Duolingo pattern)

- Hypothesis: [specific change] will cause [measurable effect] because [research-based reason].
- Primary metric (exactly one) + expected lift; secondary metrics; guardrails.
- Success looks like: ship/kill thresholds decided now — "prevents drama later" and tests the team's product intuition.
- Evidence (Confidence in ICE): attach the impact chart or cohort comparison; flat evidence lowers priority.
- Teams to inform first (domain reviews — e.g., pedagogy review for learning-content changes).
- Owner + read date (prevents forgotten experiments) + which capacity slot this occupies.

The one-pager is the defense against spaghetti testing while keeping experiment creation cheap. Duolingo pairs it with training + templates so anyone can run tests without a data-team bottleneck, plus an automated morning report of all running experiments.

## 2. Power analysis walkthrough

Inputs: baseline (from history, filtered to the eligible population), minimum meaningful effect (smallest lift worth shipping — smaller is not worth the sample), confidence 95%, power 80%.
Output: required units per variant → divide by daily eligible traffic → duration. Run >= 2 business cycles regardless. If duration comes out at months, the change is too timid for your traffic: make a bolder variant instead ("match boldness to traffic").

## 3. Exposure rules (assign ≠ expose)

Duolingo's per-user experiment object separates `destiny` (deterministic precomputed bucket = sticky assignment) from `treated` (exposure actually logged at the treatment context). Rules derived:

- Fire exposure via the SDK's tracking callback at the point the variant is rendered.
- Route-level flags may resolve in a route loader — the surface is certain to render and platforms dedup (experiment, variation) pairs.
- Item-level experiments (content a user may never open) must evaluate inside the component that renders the item; never at loader/prefetch time.
- Offline clients queue exposures and flush on reconnect.
- Sticky bucketing is mandatory when an experience spans sessions (a learner must not switch variants mid-course); hash on a stable user id shared across platforms so web and app show the same variant.

## 4. Statistics notes

- **Bayesian** (default in GrowthBook): results as "X% chance variant B is better, expected lift Y%" — easy to communicate; priors help small samples but bad priors skew them; still not immune to peeking.
- **Frequentist**: p-values, familiar audit trail; enable **sequential testing** if continuous monitoring with early stopping is needed.
- Shared regardless of school: random assignment integrity, pre-set thresholds, power analysis, SRM checks, no metric shopping (primary declared first; 20 metrics fished = false positives).
- **CUPED**: uses pre-experiment data per user to reduce variance (Microsoft: ~+20% traffic equivalent). Needs history — useless for brand-new users.
- **SRM**: a 50/50 split arriving as 55/45 means assignment is broken; results are void no matter how pretty.

## 5. Program design

- **Foundation vs experiment** (Dailymotion): big releases go through multiple staged tests — backlash check (guardrails only) → stabilize → optimize parts (one primary each) → ship & monitor post-release. Their in-house recommendation engine took 6 months and 5 sequential tests before full release.
- **Capacity** (Monzo): eligible traffic / sample per test = max concurrent. Mutual exclusivity via platform namespaces; never share control groups manually. Cost of a test includes the waiting, the reading, and the forgone alternative test.
- **Culture** (Duolingo): every result — especially losses — shared in a common channel and archived searchably; "no shame" failures teach the most; D1 retention 13% → 55% came from a decade of compounding small wins, not one big bet.
- **When not to test**: pre-PMF products, obviously-right changes (ship + non-inferiority at most), brand identity, regulated/ethical constraints (education: never split a classroom — cluster randomize by class/school), and teams that will override data with opinion anyway (fix alignment first).

## 6. Reading and acting

1. Confirm design ran as planned; sample reached; SRM clean.
2. Metric windows fully elapsed (7-day metrics wait 7 days past last exposure).
3. Guardrails and segments before celebration; segment cuts include platform, plan, and behavioral persona.
4. Practical significance: lift vs build-and-maintain cost.
5. Ship / kill / iterate — and write the learning down either way. A losing test that corrects the team's intuition before a big bet is cheap tuition.
