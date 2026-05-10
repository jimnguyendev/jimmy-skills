---
name: data-value-patterns
description: Use this skill when the user needs to increase the analytical value of data through enrichment, decoration, aggregation, sessionization, or delivery ordering. Apply it for joins with reference data, session building, feature or metric aggregation, hidden technical metadata, or choosing between distributed and local aggregation, even if the user does not explicitly say "data value patterns."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Value Patterns

Use this skill when raw events are not yet useful enough for analytics, product decisions, or downstream serving.

## Default procedure

1. Identify what is missing from the raw data:
   - business context
   - technical context
   - summarized measures
   - user sessions
   - stable ordering
2. Choose the smallest pattern that adds the missing value.
3. Keep raw lineage visible so debugging is still possible.
4. Favor reusable intermediate entities over one-off dashboard-specific transforms.

## Pattern guide

- Add static reference context: static joiner (use SCD Type 2 joins when reference data has historical changes)
- Join two moving streams: dynamic joiner
- Keep raw plus computed values together: wrapper
- Add hidden job metadata: metadata decorator
- Aggregate across distributed data: distributed aggregator (use salting to mitigate data skew on hot keys)
- Aggregate where keys are already co-located: local aggregator
- Build sessions in batch: incremental sessionizer
- Build sessions in streaming: stateful sessionizer
- Preserve delivery order to fragile sinks: bin pack or FIFO orderer

## Defaults

- Use static joins unless both sides are genuinely streaming.
- Use local aggregation when partitioning or storage layout already matches the group key.
- Use incremental sessionizer for hourly or daily batch session facts.
- Use stateful sessionizer only when low-latency sessions are required.
- Keep technical metadata hidden from business-facing marts unless users explicitly need it.

## Gotchas

- Sessionization is forward-dependent; replaying one partition can force replay of later partitions too.
- A wide aggregate is not automatically reusable. Keep the grain and consumer clear.
- Hidden metadata is valuable for operations, but do not leak it into business definitions.
- Ordering guarantees for delivery do not replace correctness checks on the target side.

## References

- Read [references/value-patterns.md](references/value-patterns.md) when the task involves enrichment, aggregation design, session facts, or ordering constraints.
