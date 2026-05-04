## Data Engineer Pack

8 skills for data platform design, stack delivery, pipeline reliability, data trust, observability, and data program leadership.

### Routing

- Use `data-engineering` when designing or debugging a warehouse workflow, analytics platform, or shared semantic model.
- Use `data-stack-delivery` when the work is tool-aware and centered on Airflow, Snowflake, dbt, Spark, Kafka, or delivery automation patterns.
- Use `data-architecture-strategy` when choosing between RDW, MDW, lakehouse, mesh, or similar platform shapes.
- Use `data-pipeline-reliability` when retries, idempotency, backfills, replay, and delivery guarantees are the main risk.
- Use `data-quality` when the main concern is contracts, reconciliation, publish gates, or schema and metric drift.
- Use `data-observability` when freshness, lag, stale dashboards, or data SLA alerting is the main problem.
- Use `data-program-leadership` when the work spans platform delivery, stakeholder alignment, sequencing, and proving data value early.

### Skills

| Skill | When to use |
|---|---|
| `data-engineering` | Pragmatic warehouse and analytics-platform design. Covers ingestion, layering (`raw`/`intermediate`/`marts`), semantic metrics, serving patterns, and cross-team ownership. |
| `data-stack-delivery` | Tool-aware modern data stack guidance for Airflow, Snowflake, dbt, Spark batch, Kafka streaming, and delivery automation. |
| `data-architecture-strategy` | Strategy-level platform choices such as RDW vs MDW vs lakehouse, centralization vs domain ownership, and how to sequence architecture change over time. |
| `data-pipeline-reliability` | Reliability patterns for retries, deduplication, idempotency, replay, backfills, ordered delivery, and failure recovery in batch or streaming pipelines. |
| `data-quality` | Trust and publish-gate patterns for contracts, validation rules, reconciliation, constraints, schema drift, and metric drift. |
| `data-observability` | Freshness, volume, schema, lineage, SLA/SLO, and alerting patterns for detecting broken or stale data systems. |
| `data-program-leadership` | Operating model guidance for leading a data program across platform, analytics, and stakeholder alignment. |
| `data-value-patterns` | Value-adding transformation patterns such as enrichment, decoration, aggregation, sessionization, and delivery ordering. |

### Relationship to Other Packs

- General engineering process → `packs/engineering/`
- Backend implementation details → `packs/backend/`
- Outcome design and outcome-oriented operating model → `packs/outcomes/`
- This pack provides the data-platform layer that sits above tool-specific implementation.
