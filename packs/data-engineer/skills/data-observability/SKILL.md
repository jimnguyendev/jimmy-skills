---
name: data-observability
description: Use this skill when the user needs to know whether data is arriving, complete, fresh, and on time. Apply it for stale marts, freshness SLAs, lag, skew, lineage, last-updated indicators, alerting, or incident triage for pipelines and warehouse datasets, even if the user does not explicitly say "observability."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Observability

Use this skill when the problem is not "is the code deployed?" but "is the data still arriving, complete, and on time?"

## Goal

Detect broken or degraded data delivery before downstream users notice.

## Working approach

1. Identify the unit of delay or completeness:
   - batch run
   - partition
   - row count
   - stream offset
   - commit version
2. Define the consumer-facing SLA first.
3. Track both freshness and volume, not just task success.
4. Alert on states that require action, not on every anomaly.

## Default procedure

1. Identify the dataset, its consumer, and the promised update time.
2. Define a formal SLA for each critical dataset: cadence, deadline, owner, escalation tiers.
3. Define one freshness signal and one volume or completeness signal.
4. Compare anomalies against the last healthy baseline, not just the last run.
5. Account for false positives: add seasonal baselines, maintenance-window suppression, and alert deduplication before enabling pages.
6. Add an owner and escalation path for every critical SLA.
7. Expose `last_updated` wherever users consume the dataset.

## Minimum observability set

- Last successful update timestamp
- Freshness SLA per critical dataset
- Volume/skew comparison to prior healthy runs
- Lag indicator for streaming or incremental consumers
- Basic lineage for high-value marts and dashboards

## Gotchas

- A green orchestrator job can still produce stale or incomplete data.
- Average lag often hides the partition or consumer that is actually failing.
- Comparing skew to the immediately previous run can create alert loops after a bad run.
- Users need dataset freshness in the interface they already use, not only in an ops dashboard.

## Reference

- Read [references/observability-patterns.md](references/observability-patterns.md) when the task involves freshness signals, lag units, skew thresholds, SLA misses, or lineage scope.
