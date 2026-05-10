---
name: data-quality
description: Use this skill when the user needs analytical data they can trust, especially for dbt tests, validation rules, metric drift, schema drift, contracts, reconciliation, or publish gates on marts and warehouse tables. Apply it when the task is to prevent bad data, detect broken business logic, or review how a dataset becomes trustworthy, even if the user does not explicitly say "data quality."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Quality

Use this skill when the user needs trustworthy datasets, not just successful pipeline runs.

## Default stance

- Prevent bad data early when possible
- Validate transformed outputs before publishing them broadly
- Make metric definitions reviewable and owned
- Combine warehouse constraints with pipeline-level tests

## Working approach

1. Identify what failure would break user trust: missing rows, wrong metric values, schema drift, stale data, or invalid business logic.
2. Decide whether the rule belongs in:
   - source/ingestion validation
   - storage constraints
   - transformation tests
   - governance and approval workflow
3. Prefer lightweight, explicit checks over opaque "quality scores".
4. Separate blocking failures from warning-only anomalies.
5. Add observation layers (offline or online) to detect issues that predefined rules do not cover.

## Default procedure

1. Define the dataset, owner, and consumer-facing risk.
2. Identify the smallest set of checks that would have caught the failure:
   - input readiness
   - source-to-target reconciliation
   - null, range, or key constraints
   - business-rule tests on transformed outputs
3. Decide which failures must block publish and which should warn only.
4. Keep metric-definition changes reviewable by the owning team.

## Recommended stack shape

- Input checks for shape, file size, row counts, or source readiness
- Warehouse constraints for nullability, basic ranges, and key integrity
- dbt or SQL tests on marts before exposure
- Metric-definition approvals for semantic changes
- Offline observer for daily distribution scans and drift detection
- Online observer for real-time quality checks inline with the pipeline

## Gotchas

- Passing ingestion does not prove that transformed business logic is correct.
- Metric drift is a quality problem, not just a dashboard problem.
- Constraints catch simple invariants, but they do not replace source-to-target reconciliation.
- If a schema or metric definition changes silently, downstream trust is already damaged even if pipelines still run.

## Reference

- Read [references/quality-patterns.md](references/quality-patterns.md) when the task involves audit gates, constraints, schema compatibility, migration strategy, or governance rules.
