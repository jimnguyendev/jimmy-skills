---
name: data-stack-delivery
description: Use this skill when the user needs tool-aware guidance for a modern data stack, especially Airflow, Snowflake, dbt, Spark batch, Kafka streaming, and delivery automation. Apply it for local learning stacks, warehouse loading, dbt project setup, batch-vs-stream decisions, or end-to-end examples from ingestion to quality gates and automation. For platform-shape decisions, prefer `jimmy-skills@data-engineering`. For retries, backfills, and replay safety as the main risk, prefer `jimmy-skills@data-pipeline-reliability`. For contracts, validation, and publish gates as the main problem, prefer `jimmy-skills@data-quality`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Stack Delivery

Use this skill when the question is not only "what architecture should we choose?" but also "how do these common data-stack tools fit together in practice?"

Official docs for Airflow, Snowflake, dbt, Spark, Kafka, and Deequ remain authoritative. Use this skill for pragmatic wiring, examples, and trade-offs.

## What this skill covers

- Airflow setup patterns for local learning and production-like orchestration
- Snowflake basics for warehouses, stages, file loading, and quick validation
- dbt project shape, build flow, tests, and team-facing model organization
- Spark batch-processing patterns and the default optimization checklist
- Kafka basics for topics, partitions, late data, and stream-processing choices
- Data quality checkpoints across Python, SQL, dbt, and Deequ
- Automation principles such as container-first delivery, slim CI/CD, and idempotent reruns

## Boundaries

- Use `jimmy-skills@data-engineering` when the main decision is platform shape, semantic metrics, marts, or multi-team ownership.
- Use `jimmy-skills@data-pipeline-reliability` when retries, replay, deduplication, or backfill safety are the primary risk.
- Use `jimmy-skills@data-quality` when the main task is designing validation rules, contracts, reconciliation, or publish gates.
- Use `jimmy-skills@data-observability` when the main task is freshness, lag, stale dashboards, or SLA alerting after delivery.

## Working approach

1. Separate learning-stack guidance from production-like guidance before suggesting tools.
2. Pick the simplest end-to-end flow that proves the data path works.
3. Make grain, idempotency, quality checks, and ownership explicit before adding more tooling.
4. Verify with SQL, tests, or consumer-facing outputs instead of trusting a green orchestrator alone.

## Default procedure

1. Identify which stage the user is working on:
   - orchestration
   - warehouse loading
   - transformation
   - batch processing
   - stream processing
   - quality gate
   - automation
2. Choose the narrowest toolchain that fits the requirement.
3. Define the input, output, and success check for that stage.
4. Add one explicit quality or correctness check before calling the stage done.
5. Document the handoff to the next stage in the stack.

## Defaults

- Prefer a lightweight local Airflow setup for learning, then move to the official or managed stack for production-like behavior.
- Start Snowflake with small warehouses, staged files, and explicit load-validation steps.
- Use `dbt build` and `dbt test` as the default transformation and publish workflow.
- Start with Spark batch before Kafka streaming unless consumer latency truly requires real time.
- Treat automation as delivery safety, CI/CD, and rerun discipline, not just cron scheduling.

## Gotchas

- A successful Airflow run does not prove the warehouse table is correct.
- `inferSchema` is convenient for demos, but explicit schemas are safer for production batch jobs.
- A dbt model that builds without tests is not a trustworthy publish gate.
- The public Week 8 automation material is directional, not a complete implementation guide.

## Reference

- Read [references/tooling-playbook.md](references/tooling-playbook.md) when the task needs practical Airflow, Snowflake, dbt, Spark, Kafka, quality, and automation examples in one path.
- Read [references/sources/README.md](references/sources/README.md) when you need the in-repo Datacamping source notes instead of external pages.
