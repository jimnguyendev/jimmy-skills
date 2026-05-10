# Platform Patterns

This reference synthesizes data-engineering principles into an actionable operating model for analytics platforms. Use it for lifecycle reasoning, modeling decisions, transformation design, serving strategy, and multi-team coordination.

---

## Lifecycle framework

### Five stages

Every data platform moves through five stages. Storage is not a discrete step -- it spans all stages.

```
Generation --> Ingestion --> Transformation --> Serving
                        |
                     Storage (spans every stage)
```

| Stage | Key questions |
|-------|--------------|
| **Generation** | Where does data originate? How often does it change? How stable is the schema? Is the upstream system reliable? |
| **Ingestion** | What latency is actually required? Batch or streaming? How to handle late-arriving data and deletes? |
| **Storage** | Hot, warm, or cold? Schema-on-write or schema-on-read? What durability and cost constraints apply? |
| **Transformation** | What business logic needs to become reusable? What grain, freshness, and quality bar apply? |
| **Serving** | Who consumes this data, through which interface, and for which decision? |

### Six undercurrents

These cross-cutting concerns apply at every stage, not as separate steps:

| Undercurrent | Scope |
|-------------|-------|
| **Security** | Access control, encryption at rest and in flight, least-privilege, PII handling |
| **Data Management** | Governance, discoverability, quality (accuracy, completeness, timeliness), metadata, lineage |
| **DataOps** | CI/CD for pipelines, monitoring, observability, incident response, blameless postmortems |
| **Data Architecture** | Trade-off analysis, design for agility, cost vs simplicity balance |
| **Orchestration** | Dependency-aware workflow coordination (not just cron scheduling), retries, backfills |
| **Software Engineering** | Production-grade code, testing, IaC, pipelines-as-code |

---

## Data maturity model

Use this to calibrate advice and architecture complexity to the organization's actual stage.

| | Stage 1: Starting | Stage 2: Scaling | Stage 3: Leading |
|---|---|---|---|
| **Team** | 1-2 people, generalists, many hats | Specialized roles (DE, analyst, architect) | Dedicated platform team, embedded analysts |
| **Practices** | Ad hoc reports, manual queries | Formal pipelines, DataOps adoption | Self-service analytics, automated quality gates |
| **Architecture** | Single DB, cron jobs, spreadsheets | Warehouse + orchestrator + dbt | Data products with SLAs, catalog, lineage |
| **Risk** | Jumping to ML before having data foundation | Chasing bleeding-edge tech instead of delivering value | Complacency -- assuming the platform is "done" |
| **DE focus** | Get buy-in, define architecture, avoid over-engineering | Formalize practices, adopt DevOps, optimize team throughput | Automate everything, governance, custom tools only for competitive advantage |

**Rule of thumb:** use managed services and off-the-shelf tools (Type A approach) at every stage. Build custom only when it creates a measurable competitive advantage (Type B approach).

---

## Modeling guidance

### Approach comparison

| Approach | Structure | Best for | Trade-offs |
|----------|----------|----------|-----------|
| **Kimball** (dimensional) | Star schema: fact tables + dimension tables | Most analytics programs at Stage 1-2 | Fast to value, easy to query; denormalized, some redundancy |
| **Inmon** (enterprise DW) | Fully normalized 3NF, top-down | Large enterprises needing single source of truth | Consistent, less redundancy; slow to deliver, heavy upfront design |
| **Data Vault 2.0** | Hubs (business keys) + Links (relationships) + Satellites (attributes + history) | Multi-source integration, audit trail, schema volatility, GDPR | Full history, additive schema; many tables, complex queries, needs marts on top |
| **Wide denormalized** | Single pre-joined table per analytical domain | Read-heavy OLAP layers (BigQuery, Snowflake, ClickHouse) | Fast scans, simple queries; not a source-of-truth layer |

**Default recommendation:** start with Kimball. Consider Data Vault when audit history, compliance, or multi-source integration dominate. Use wide tables as a serving optimization, not as the modeling layer.

### Grain definition

Grain answers: "what does one row in this fact table represent?" Define it before designing anything else.

```sql
-- Grain: one row = one exam attempt by one user
CREATE TABLE fact_exam_attempt (
  attempt_id      BIGINT PRIMARY KEY,
  user_id         BIGINT NOT NULL,
  course_id       BIGINT NOT NULL,
  attempt_date    DATE NOT NULL,
  score           DECIMAL(5,2),
  duration_sec    INT,
  is_passed       BOOLEAN
);

-- Grain: one row = one user-day aggregation
CREATE TABLE fact_user_daily (
  user_id         BIGINT NOT NULL,
  activity_date   DATE NOT NULL,
  questions_attempted INT,
  accuracy_rate   DECIMAL(5,4),
  time_spent_sec  INT,
  streak_day      INT,
  PRIMARY KEY (user_id, activity_date)
);
```

**Principle:** choose the lowest meaningful grain. You can always aggregate up; you cannot drill down below the stored grain.

### One Big Table anti-pattern

Dumping every column into a single table without a clear grain leads to ambiguous row semantics, runaway width, and conflicting update cadences. Use wide tables intentionally for a defined serving purpose, not as a substitute for modeling.

### Slowly Changing Dimensions (SCD)

| Type | Behavior | When to use |
|------|---------|-------------|
| Type 1 | Overwrite -- no history | Typo corrections, email changes |
| Type 2 | New row with `valid_from / valid_to` -- full history | Subscription tier, pricing, any attribute that affects historical analysis |
| Type 3 | Add `previous_value` column -- one step of history | Rarely used |

### Metrics layer

Define business metrics once in code. If two teams can answer the same question with different SQL, the platform is not ready.

```yaml
# dbt metrics / MetricFlow style
metrics:
  - name: completion_rate
    description: "Fraction of enrolled users who finished all lessons in a course"
    type: ratio
    numerator: count_distinct(user_id) where is_completed = true
    denominator: count_distinct(user_id) where is_enrolled = true
    grain: course_id, cohort_week

  - name: active_users_7d
    description: "Distinct users with at least one session in the trailing 7 days"
    type: count_distinct
    column: user_id
    filter: activity_date >= current_date - 7
```

Metrics to define early: `completion_rate`, `active_users_Nd`, `churn_rate_30d`, `trial_to_paid_conversion`, `avg_score_improvement`.

---

## Transformation patterns

### ETL vs ELT

| | ETL | ELT |
|---|---|---|
| Transform location | Outside the warehouse | Inside the warehouse (SQL) |
| Raw data preserved | Often not | Yes -- enables reprocessing when logic changes |
| Current default | Legacy on-prem | **Modern standard** for cloud warehouses |

**Default:** ELT. Load raw data first, transform with SQL in the warehouse.

### Staging --> intermediate --> marts

```
sources (raw) --> staging --> intermediate --> marts
```

| Layer | Purpose | Naming convention |
|-------|---------|-------------------|
| **Staging** | 1:1 with source tables. Clean, rename, cast types. No business logic. | `stg_<source>_<table>` |
| **Intermediate** | Reusable business entities, multi-table joins, complex logic | `int_<entity>_<verb>` |
| **Marts** | Team-facing or decision-facing outputs | `mart_<domain>_<topic>` |

```sql
-- staging: clean and rename only
-- stg_app__exam_attempts.sql
SELECT
  id                AS attempt_id,
  user_id,
  course_id,
  CAST(created_at AS DATE) AS attempt_date,
  score,
  duration_seconds  AS duration_sec,
  score >= passing_threshold AS is_passed
FROM {{ source('app_db', 'exam_attempts') }}

-- intermediate: reusable business logic
-- int_user_daily_activity.sql
SELECT
  user_id,
  attempt_date AS activity_date,
  COUNT(*)          AS questions_attempted,
  AVG(CASE WHEN is_passed THEN 1.0 ELSE 0.0 END) AS accuracy_rate,
  SUM(duration_sec) AS time_spent_sec
FROM {{ ref('stg_app__exam_attempts') }}
GROUP BY user_id, attempt_date

-- mart: team-facing output
-- mart_learning_metrics.sql
SELECT
  d.activity_date,
  COUNT(DISTINCT d.user_id) AS active_users,
  AVG(d.accuracy_rate)      AS avg_accuracy,
  SUM(d.time_spent_sec)     AS total_time_sec
FROM {{ ref('int_user_daily_activity') }} d
GROUP BY d.activity_date
```

### Update patterns

| Pattern | Mechanism | When to use |
|---------|----------|-------------|
| **Full refresh** | Truncate + reload | Small tables, complex logic where incremental is not worth it |
| **Incremental** | Process only rows newer than a watermark (`updated_at` or event timestamp) | Large tables, append-heavy data |
| **Upsert / merge** | Insert if new, update if existing | Source data with retroactive corrections |

**Idempotency** is non-negotiable: rerunning a pipeline must produce the same result regardless of how many times it executes.

### SQL patterns for analytics

**CTEs for readability:**

```sql
WITH daily_scores AS (
  SELECT user_id, attempt_date, AVG(score) AS avg_score
  FROM {{ ref('stg_app__exam_attempts') }}
  GROUP BY user_id, attempt_date
),
with_lag AS (
  SELECT *,
    LAG(avg_score) OVER (PARTITION BY user_id ORDER BY attempt_date) AS prev_score
  FROM daily_scores
)
SELECT *, avg_score - prev_score AS score_delta
FROM with_lag
```

**Window functions** for running totals, ranking, and change detection are the workhorses of analytical SQL. Prefer them over self-joins.

---

## Serving patterns

### Serving decision matrix

| Pattern | Audience | Latency | Tools | When to use |
|---------|---------|---------|-------|-------------|
| **BI / dashboards** | Leadership, product, marketing | Hours-days (daily refresh) | Metabase, Looker, Tableau, Redash | Strategic decisions, weekly reviews |
| **Embedded analytics** | End users inside the product | Seconds | Custom UI reading from marts or API | Competitive advantage in B2C/B2B -- learner progress, score trends |
| **Operational analytics** | Ops, on-call, support | Seconds-minutes | Grafana, Datadog | Real-time alerting, platform health, error spike detection |
| **Reverse ETL** | CRM, marketing, support tools | Minutes-hours | Hightouch, Census, or custom Airflow jobs | Warehouse insights must drive action in external tools |
| **ML feature serving** | ML models (training + inference) | Varies | Feature stores (Feast, Tecton) | Only after analytics foundation is stable |
| **Federation / data sharing** | Partner orgs, external consumers | Varies | Snowflake data sharing, BigQuery authorized views | Cross-org data exchange without copying |

### Self-service analytics

Requires: trusted data quality, a catalog or wiki describing available datasets, and data literacy in the organization. Before enabling self-service, ensure core metrics are defined in the semantic layer so ad-hoc queries do not reinvent definitions.

### Multitenancy in embedded analytics

When serving analytics inside a product, enforce strict row-level security. A data leak between tenants (user A sees user B's data) is a security and reputation catastrophe.

---

## Metrics framework

### Four-tier hierarchy

```
Tier 1: North Star (Business Health)
  Revenue, MAU, Churn Rate, LTV
  --> "Is the business healthy?"

Tier 2: Product Metrics (Feature Effectiveness)
  Completion Rate, DAU/MAU stickiness, Session Duration, Feature Adoption
  --> "Is the product solving the user's problem?"

Tier 3: Quality / Health (Technical + UX)
  Error Rate per Active User, CSAT, Latency, Bug Report Rate
  --> "Is the experience stable?"

Tier 4: Leading Indicators (Predict Future)
  Streak count, Time-to-first-value, Onboarding completion, Engagement cohort
  --> "Will users stay?"
```

### Six-step feature decision framework

Before building any feature, walk through this sequence:

1. **Which metric will improve?** If unknown, stop. Do not build.
2. **What is the current value?** If unknown, instrument and collect for 2-4 weeks first.
3. **What is the target value?** Must be specific: "completion rate 35% --> 45%."
4. **Effort vs impact?** Low effort + high impact = do it now. High effort + low impact = skip.
5. **Can the hypothesis be validated with existing data?** Check before building.
6. **After shipping, did the metric hit the target?** If not, iterate or rollback.

**Core principle:** metrics before features. Every metric must answer: "If this number changes, what will I do differently?" If it cannot, it is a vanity metric.

### Event schema design

```
Event: answer_submitted
  user_id, question_id, skill, is_correct, time_spent_seconds,
  attempt_number, question_type, difficulty_tag

Event: lesson_started / lesson_completed
  user_id, lesson_id, course_id, started_at, completed_at

Event: test_started / test_completed
  user_id, test_id, test_type (mock/practice/diagnostic),
  is_timed, total_score, section_scores{}
```

### Three-layer aggregation pipeline

```
Layer 1: Raw events (immutable, append-only)
  --> answer_submitted, lesson_completed, test_completed

Layer 2: Per-user daily aggregates (dbt intermediate)
  --> questions_attempted, accuracy_rate, time_spent_total,
      skill_breakdown, streak_day_count

Layer 3: Business-ready marts
  --> mart_learner_progress (estimated band, gap to target, strongest/weakest skill)
  --> mart_content_effectiveness (completion rate, drop-off rate, difficulty calibration)
  --> mart_business_metrics (DAU/MAU, stickiness, churn, conversion, LTV)
```

---

## Multi-team strategy

### Data mesh lite

For organizations with several product teams but limited data headcount, full data mesh (decentralized infrastructure per domain) is premature. Instead:

| Responsibility | Owner |
|---------------|-------|
| **Metric definitions and business rules** | Domain teams (product, content, marketing) |
| **Pipeline implementation and operations** | Central platform team |
| **Data quality gates and SLAs** | Shared -- domain proposes, platform enforces |
| **Schema changes** | Domain proposes, platform reviews for downstream impact |
| **Warehouse infrastructure** | Central platform team |

This is domain ownership of **meaning**, centralized ownership of **execution**.

### Five governance rules

1. Every metric has exactly one canonical definition in the semantic layer.
2. Every mart has a named owner and a freshness SLA.
3. Schema changes require downstream-impact review before deploy.
4. PII columns are tagged and access-controlled at the warehouse level.
5. Pipeline failures page an on-call human, not just log a warning.

### Power-user enablement

Train domain analysts to write read-only SQL against marts and curated views. Provide a data wiki or catalog documenting available datasets, grain, freshness, and known caveats. This reduces bottleneck on the platform team without opening raw data to untrained users.

---

## Architecture defaults (quick reference)

| Decision | Default | Override when |
|----------|---------|--------------|
| Batch vs streaming | Batch | Explicit sub-minute latency requirement for operations or product behavior |
| ETL vs ELT | ELT | Legacy on-prem system with no analytical warehouse |
| OLTP vs OLAP | Separate stores | Tiny dataset where a single DB handles both without contention |
| Modeling approach | Kimball star schema | Audit/compliance needs (Data Vault) or enterprise-wide 3NF (Inmon) |
| Orchestration | DAG-aware orchestrator (Airflow, Dagster) | Trivially small workload that a single cron job can handle |
| Semantic layer | dbt metrics or equivalent | Organization already uses a BI tool with built-in semantic layer (Looker LookML) |
| Self-service | Marts + curated views | Data literacy and quality are not yet sufficient |
