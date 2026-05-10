# Quality Patterns

Use these patterns to build confidence in analytical data products. Three categories: quality enforcement (prevent bad data), schema consistency (prevent breaking changes), and quality observation (detect new issues before consumers do).

## Audit-Write-Audit-Publish (AWAP)

AWAP adds two audit gates around transformation: one before and one after.

### Input audit

Lightweight checks before transformation begins:

- **File size and format**: reject empty or truncated files early.
- **JSON / CSV validity**: parse for malformed lines.
- **Row count bounds**: source row count within expected range.
- **Schema shape match**: columns and types align with the contract.

### Output audit

Validate transformed data before it reaches consumers:

- **Business rule validation**: metric values within sane bounds.
- **Nullability checks**: required columns contain no NULLs.
- **Source-to-target reconciliation**: output row count matches input within tolerance.
- **Metric bounds**: aggregated metrics do not deviate beyond a threshold from recent history.

### Publish semantics

| Mode | Behaviour |
|------|-----------|
| **All-or-nothing** | Pipeline halts on failure. Safest default. |
| **Data dispatching** | Valid records to consumers; invalid to dead-letter storage. |
| **Non-blocking** | Data published but annotated with quality warnings. |

### Airflow AWAP example

```python
@dag(schedule="@daily")
def awap_pipeline():
    @task
    def audit_input(**ctx):
        if os.path.getsize(source_path(ctx)) < MIN_FILE_BYTES:
            raise ValueError("File too small")
        with open(source_path(ctx)) as f:
            for i, line in enumerate(f, 1):
                json.loads(line)  # raises on invalid JSON

    @task
    def audit_output(**ctx):
        df = pd.read_csv(output_path(ctx))
        for col in ["visit_id", "event_time", "user_id"]:
            if df[col].isnull().any():
                raise ValueError(f"NULLs in {col}")

    audit_input() >> transform() >> audit_output() >> publish()
```

### Spark Structured Streaming AWAP

```python
visits = (spark.readStream.format("delta").table("staging.visits")
    .withColumn("is_valid", row_validation_expr))

def audit_and_write(batch_df, batch_id):
    invalid = batch_df.filter(~col("is_valid")).count()
    if invalid > MAX_INVALID:
        raise ValueError(f"Batch {batch_id}: {invalid} invalid rows")
    batch_df.filter(col("is_valid")).write.mode("append").saveAsTable("prod.visits")

visits.writeStream.trigger(processingTime="30 seconds") \
    .foreachBatch(audit_and_write).start()
```

### dbt as AWAP

Models plus tests form the publish gate. If any test fails, `dbt build` exits non-zero and downstream exposures are not refreshed.

```yaml
models:
  - name: mart_daily_active_users
    columns:
      - name: user_id
        tests: [not_null, unique]
      - name: activity_date
        tests: [not_null]
      - name: session_count
        tests:
          - not_null
          - dbt_utils.accepted_range: {min_value: 1}
    tests:
      - dbt_utils.recency: {datepart: day, field: activity_date, interval: 1}
```

---

## Constraints Enforcer

Delegate validation to the storage layer or serialization format. Declarative, no custom code per pipeline.

### Constraint types

| Type | Purpose | Example |
|------|---------|---------|
| **Type** | Column values share a single type | `event_time TIMESTAMP` |
| **Nullability** | Prevent missing required values | `user_id STRING NOT NULL` |
| **Value** | Validate ranges and business rules | `CHECK (amount >= 0)` |
| **Integrity** | Enforce relationships and uniqueness | `UNIQUE`, `FOREIGN KEY` |

### Delta Lake constraints

```sql
CREATE TABLE visits (
    visit_id STRING NOT NULL,
    event_time TIMESTAMP NOT NULL,
    user_id STRING NOT NULL
) USING delta;

ALTER TABLE visits ADD CONSTRAINT event_time_not_future
    CHECK (event_time < NOW() + INTERVAL '1 SECOND');
```

### ClickHouse constraints

```sql
CREATE TABLE visits (
    visit_id String, event_time DateTime, user_id String,
    completion_rate Float64, attempt_count UInt32
) ENGINE = MergeTree()
ORDER BY (event_time, visit_id);

ALTER TABLE visits ADD CONSTRAINT rate_range CHECK completion_rate BETWEEN 0 AND 1;
ALTER TABLE visits ADD CONSTRAINT positive_attempts CHECK attempt_count >= 0;
ALTER TABLE visits ADD CONSTRAINT valid_user CHECK length(user_id) > 0;
```

### Protobuf + protovalidate at ingestion

```protobuf
message Visit {
    string visit_id = 1 [(buf.validate.field).string.min_len = 5];
    google.protobuf.Timestamp event_time = 2 [
        (buf.validate.field).timestamp.lt_now = true,
        (buf.validate.field).required = true
    ];
    string page = 4 [(buf.validate.field).cel = {
        message: "Page cannot end with .html"
        expression: "this.endsWith('html') == false"
    }];
}
```

### All-or-nothing vs advisory

Database constraints are all-or-nothing: one bad row rejects the entire transaction. Strategies:

- Route invalid records to a dead-letter table before the constrained insert.
- Use staging tables without constraints, validate in SQL, then insert only clean rows.
- For advisory checks (warn but do not block), use AWAP output audits instead.

---

## Schema Compatibility Enforcer

Prevent upstream schema changes from breaking downstream consumers.

### Compatibility modes

| Mode | Allowed changes | Semantics |
|------|----------------|-----------|
| **Backward** | Delete fields, add optional fields | New consumer reads old data |
| **Forward** | Add fields, delete optional fields | Old consumer reads new data |
| **Full** | Add or delete optional fields only | Both directions compatible |
| **None** | Any change allowed | No compatibility guarantee |

### Transitive vs non-transitive

| Variant | Checks against | Use when |
|---------|---------------|----------|
| **Non-transitive** | Previous version only | Small teams, infrequent changes |
| **Transitive** | All previous versions | Many consumers on different versions |

```text
v0: order_id LONG REQUIRED
v1: order_id LONG REQUIRED, amount DOUBLE DEFAULT 0.0   -- backward compat with v0
v2: order_id LONG REQUIRED, amount DOUBLE REQUIRED       -- backward compat with v1

Non-transitive: v1 <-> v2 OK
Transitive:     v0 <-> v2 FAIL (v0 missing required amount)
```

### Kafka Schema Registry example

```json
{
    "type": "record", "name": "Visit", "namespace": "com.example.events",
    "fields": [
        {"name": "visit_id", "type": "string"},
        {"name": "event_time", "type": {"type": "int", "logicalType": "timestamp-millis"}},
        {"name": "page", "type": ["null", "string"], "default": null}
    ]
}
```

```bash
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD_TRANSITIVE"}' \
  http://schema-registry:8081/config/visit-value
```

### Breaking vs non-breaking changes

| Change | Backward | Forward | Full |
|--------|----------|---------|------|
| Add optional field with default | Safe | Safe | Safe |
| Add required field without default | Breaks | Safe | Breaks |
| Remove optional field | Safe | Breaks | Breaks |
| Remove required field | Breaks | Breaks | Breaks |
| Rename field | Breaks | Breaks | Breaks |
| Change field type | Breaks | Breaks | Breaks |

---

## Schema Migrator

Safe schema evolution through a grace period: add new, let consumers adapt, remove old.

### Grace period pattern

```text
Phase 1 - Add:        ADD COLUMN referral; producers populate both old and new
Phase 2 - Use:         Consumers migrate to read new column
Phase 3 - Deprecate:   Log warnings when old column is accessed
Phase 4 - Remove:      DROP COLUMN from_page
```

```sql
-- WRONG: breaks consumers immediately
ALTER TABLE visits RENAME COLUMN from_page TO referral;

-- RIGHT: grace period
ALTER TABLE visits ADD COLUMN referral VARCHAR(25);
UPDATE visits SET referral = from_page;
-- wait for consumers to migrate, then drop
```

### Dual-write / dual-read

During the grace period, producers write to both old and new columns. Consumers read whichever they support. For event streams, emit events in both formats until all consumers upgrade.

### Backfill strategy for metric logic changes

1. Add the new metric column alongside the old one.
2. Backfill historical data with the new logic.
3. Run both in parallel for 1-2 weeks.
4. Compare old vs new to confirm the change is correct.
5. Deprecate the old metric after stakeholder sign-off.

### Deprecation timelines

| Change type | Grace period |
|-------------|-------------|
| Add optional column | Immediate |
| Rename column | 2-4 weeks dual-write |
| Change column type | 2-4 weeks parallel columns |
| Remove column | 2-4 weeks after zero downstream usage confirmed |
| Metric logic change | 1-2 weeks parallel, then stakeholder approval |

### Version coexistence

When multiple schema versions coexist in the same table, add a `schema_version` column. Consumers filter or branch logic based on the version.

---

## Quality Observation Patterns

Auditing is blocking (pipeline stops on failure). Observation is non-blocking (detects issues, alerts team, pipeline continues). Use observation to catch problems that predefined rules miss.

### Offline Observer

A separate job that runs independently from the main pipeline, analyzing processed records for statistics, distributions, and outliers.

**When to use**: monitoring is needed but a delay between data landing and insight is acceptable (e.g. hourly pipeline, daily quality scan).

**What it checks**: row count trends, NULL rates per column, value distribution shifts (mean, median, percentiles), outlier detection, schema drift.

#### SQL monitoring queries

```sql
-- Daily row counts and NULL rates
INSERT INTO visits_monitoring (observation_date, total_rows,
    null_event_time_rate, null_user_id_rate, avg_duration, p95_duration)
SELECT
    CURRENT_DATE,
    COUNT(*),
    ROUND(SUM(CASE WHEN event_time IS NULL THEN 1 ELSE 0 END)::DECIMAL / COUNT(*), 4),
    ROUND(SUM(CASE WHEN user_id IS NULL THEN 1 ELSE 0 END)::DECIMAL / COUNT(*), 4),
    AVG(duration_seconds),
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_seconds)
FROM visits_output
WHERE created_at >= CURRENT_DATE - INTERVAL '1 day';

-- Distribution shift detection
SELECT observation_date, total_rows,
    LAG(total_rows) OVER (ORDER BY observation_date) AS prev_rows,
    ROUND(ABS(total_rows - LAG(total_rows) OVER (ORDER BY observation_date))::DECIMAL
        / NULLIF(LAG(total_rows) OVER (ORDER BY observation_date), 0), 4) AS change_pct
FROM visits_monitoring ORDER BY observation_date DESC LIMIT 14;
```

**Trade-offs**: non-blocking but delayed insight; scanning a full day can be expensive (consider sampling).

### Online Observer

Observation embedded in the data pipeline, providing real-time quality checks.

**When to use**: the cost of delayed detection is high (dashboard metrics consumed within minutes).

#### Sequential vs parallel

| Approach | Flow | Trade-off |
|----------|------|-----------|
| **Sequential** | Transform -> Observe -> Load | Higher latency, safer |
| **Parallel** | Transform -> (Observe &#124;&#124; Load) | Lower latency, riskier |

#### Airflow online observer

```python
wait_for_data >> transform >> record_observation_state >> insert_observations \
    >> mark_pipeline_success  # trigger_rule='all_done'
```

Set `trigger_rule='all_done'` so observation failures alert but do not block delivery.

#### When to block vs warn

- **Block**: customer-facing metric where a wrong value causes real harm.
- **Warn**: internal metric consumed by analysts who can tolerate a next-day correction.

---

## Governance Rules

Keep governance narrow, operational, and enforceable through automation.

### Rule 1: every mart has an owner

```yaml
models:
  - name: mart_revenue
    meta:
      owner: finance-data-team
      slack_channel: "#data-finance"
```

### Rule 2: every metric has one canonical definition

The canonical definition lives in one place (a dbt model, a metrics layer, or a documented SQL query). All dashboards reference that single source.

### Rule 3: access control per team

Grant `SELECT` on marts to consuming teams. Restrict raw and intermediate layers to the data team. Review grants in CI when permissions files change.

### Rule 4: data freshness monitoring and SLA alerts

```yaml
sources:
  - name: raw_events
    freshness:
      warn_after: {count: 2, period: hour}
      error_after: {count: 6, period: hour}
    loaded_at_field: _loaded_at
```

### Rule 5: metric definition changes need owner approval

Enforce with CODEOWNERS on metric model files, CI checks that flag metric SQL changes without owner approval, and versioned definitions in dbt `schema.yml`.

---

## Expanded Quality Checklist

Before exposing a dataset to users:

| Category | Check | Example |
|----------|-------|---------|
| **Grain** | Grain is explicit and documented | "One row per user per day" |
| **Ownership** | Owner team and contact are known | `meta.owner` in schema.yml |
| **Freshness** | Freshness SLA documented and monitored | `warn_after: 2 hours` |
| **Null tests** | Required columns have not_null tests | `tests: [not_null]` on user_id |
| **Key tests** | Primary key tested for uniqueness | `tests: [unique]` on composite key |
| **Reconciliation** | Source-to-target row count check | Input vs output within 1% |
| **Metric bounds** | Core metrics have range tests | DAU > 0, rate between 0 and 1 |
| **Schema contract** | Column names and types pinned | dbt contract or schema test |
| **Observation** | Offline or online observer running | Daily NULL rate monitoring |
| **Metric versioning** | Definitions versioned or reviewable | Owner approval via CODEOWNERS |
