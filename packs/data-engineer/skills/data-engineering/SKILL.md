---
name: data-engineering
description: Use this skill when the user needs to design or debug an analytics platform, warehouse workflow, or team-facing data model. Apply it for raw/intermediate/mart layering, facts and dimensions, semantic metrics, serving patterns, reverse ETL, and how multiple teams should share one data platform, even if the user does not explicitly say "data engineering." For tool-aware workflows across Airflow, Snowflake, dbt, Spark, or Kafka, prefer `jimmy-skills@data-stack-delivery`. For retry safety, backfills, and idempotency, prefer `jimmy-skills@data-pipeline-reliability`. For RDW vs MDW vs lakehouse decisions, prefer `jimmy-skills@data-architecture-strategy`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Engineering

Use this skill when the user is building or fixing a data platform, analytics stack, or warehouse-backed reporting workflow.

## What this skill covers

- Reasoning through the full data engineering lifecycle (generation, ingestion, storage, transformation, serving) and the six undercurrents (security, data management, DataOps, data architecture, orchestration, software engineering)
- Calibrating architecture complexity to the organization's data maturity stage
- Designing dbt-style staging/intermediate/mart layers with explicit grain and update patterns
- Picking data models for analytics workloads (Kimball, Inmon, Data Vault, wide tables) with concrete trade-offs
- Defining metrics before building dashboards or features, using a four-tier hierarchy and six-step decision framework
- Choosing serving patterns: BI, embedded analytics, operational analytics, reverse ETL, ML feature serving
- Balancing centralized execution with domain ownership ("data mesh lite")

## Boundaries

- Use `jimmy-skills@data-pipeline-reliability` when the main problem is retries, duplicates, backfills, replay behavior, overwrite vs merge, or ordered delivery.
- Use `jimmy-skills@data-architecture-strategy` when the main problem is choosing between RDW, MDW, lakehouse, data fabric, or data mesh.
- Use `jimmy-skills@data-quality` when the main problem is publish gates, contracts, reconciliation, or schema drift.
- Use `jimmy-skills@data-observability` when the main problem is freshness, lag, skew, stale dashboards, or SLA alerting.
- Use `jimmy-skills@data-stack-delivery` when the main question is how Airflow, Snowflake, dbt, Spark, Kafka, and delivery automation fit together in practice.

## Working approach

1. Start from the decision the user needs to enable, not from the tool.
2. Identify the lifecycle stage involved: generation, ingestion, storage, transformation, or serving.
3. Prefer boring architecture:
   - Batch before streaming unless latency requirements are explicit
   - ELT before bespoke ETL when a warehouse can handle transforms
   - Centralized warehouse + domain-reviewed definitions before pure data mesh
4. Make grain, ownership, and freshness explicit before writing pipelines.
5. Treat every mart, metric, or dashboard as a data product with an owner, SLA, and quality checks.

## Default procedure

1. Define the business question or decision first.
2. Write down the grain, freshness target, and owner.
3. Choose the simplest ingestion mode that satisfies the latency requirement.
4. Keep source-shaped data in `raw`, reusable business entities in `intermediate`, and team-facing outputs in `marts`.
5. Define shared metrics once before building dashboards, alerts, or feature logic.
6. Validate that the serving layer matches the consumer:
   - BI for reviews and planning
   - embedded analytics for product experiences
   - operational analytics for fast response
   - reverse ETL for action in external tools

## Default architecture bias

- `raw` captures source-shaped data with minimal logic
- `intermediate` expresses reusable business entities and joins
- `marts` serve a specific team or decision
- Shared dimensions and semantic metrics are defined once and reused

## Heuristics

- If the question is "should this be real-time?", challenge it. Most analytics use cases should start with hourly or daily batch.
- If analytics queries are hitting an OLTP database, move them to an OLAP store.
- If metrics differ across teams, add a metrics/semantic layer before adding more dashboards.
- If the organization is small or mid-sized, borrow domain ownership ideas without adopting a full data mesh.
- If the user is in edtech or B2C learning, prioritize completion, retention, engagement, and content-quality feedback loops.

## Gotchas

- A successful pipeline run is not the same as a trustworthy dataset. Quality and freshness checks still need to pass.
- Do not design marts before the row grain is explicit. Most reporting errors start there.
- Do not let each dashboard redefine core metrics such as `active_user`, `completion_rate`, or `churn_rate`.
- For small and mid-sized teams, "data mesh" usually means domain review of definitions plus centralized platform execution, not decentralized infrastructure.

## References

- Read [references/platform-patterns.md](references/platform-patterns.md) when the task requires lifecycle framing, modeling trade-offs, serving choices, or multi-team platform strategy.
