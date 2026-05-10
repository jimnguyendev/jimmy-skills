# Observability Patterns

Two pattern groups: **detection** (flow interruption, skew, lag, SLA miss) spots problems now; **tracking** (dataset tracker, fine-grained tracker) maps dependencies for root-cause analysis later.

---

## Flow Interruption Detector

Detect when expected data stops arriving before downstream consumers notice.

### Detection strategies by data pattern

**Continuous data** -- check `max(updated_at)` against a freshness threshold:

```sql
-- PostgreSQL: seconds since last committed transaction on the table
SELECT
  CAST(EXTRACT(EPOCH FROM NOW()) AS INT) AS "time",
  CAST(EXTRACT(EPOCH FROM NOW() - MAX(pg_xact_commit_timestamp(xmin)))
       AS INT) AS seconds_since_last_update
FROM analytics.visits_flattened;
-- Alert when seconds_since_last_update > threshold (e.g. 3600 = 1 hour)
```

**Irregular data** -- baseline expected gap from historical data. Use a wider window than the longest observed healthy gap to avoid false positives on intentionally sporadic data. Alert only when the no-data period exceeds the accepted threshold; a single missed window is noise.

**Batch / data-at-rest** -- check three layers:

| Layer | What to check | Example |
|-------|---------------|---------|
| Metadata | Job completion flag | Airflow task state, dbt run result |
| Storage | New files or partitions | S3 listing, Hive partition count |
| Data | Row count change | `SELECT COUNT(*) WHERE date = CURRENT_DATE` |

Always verify at the data layer. Schema evolution registers as a metadata change, and compaction creates new files without adding data. A row-count check between two consecutive executions is the safest signal: if the count is unchanged, the pipeline stalled.

```sql
-- Batch row-count interruption: no new data if count is identical across two runs
SELECT COUNT(*) AS row_count
FROM raw.visits
WHERE partition_date = CURRENT_DATE;
-- Store result per run; alert when current == previous
```

### Prometheus freshness metric

```python
from prometheus_client import Gauge, push_to_gateway
import time

freshness = Gauge('data_last_successful_update_epoch_seconds',
                  'Unix epoch of last successful write',
                  ['dataset', 'pipeline'])
freshness.labels(dataset='visits', pipeline='pg_to_ch').set(time.time())
push_to_gateway('localhost:9091', job='freshness_tracker',
                registry=freshness._registry)
```

Set an alert rule that fires when `time() - data_last_successful_update_epoch_seconds > SLA_seconds`.

### False positive mitigation

- **Holidays/weekends**: Suppress alerts during known zero-traffic windows.
- **Planned maintenance**: Pause detectors during scheduled downtime.
- **Threshold tuning**: Base on historical P95 gap, not average. Review quarterly.
- **Compaction awareness**: Distinguish file mtime change (compaction) from content change (new data). Prefer row-count or commit-based checks over mtime.

---

## Skew Detector

Detect abnormal volume or distribution shifts indicating incomplete data.

### Window-to-window comparison

Compare against a **healthy baseline**, not the previous run. If run N was skewed and you compare N+1 against it, both look normal and the cascading alert never fires — the "fatality loop." Tag each run as healthy or unhealthy; always compare against the last healthy-tagged run.

```python
# Airflow: compare file sizes between DAG runs
def compare_volumes():
    context = get_current_context()
    prev = DagRun.get_previous_dagrun(context['dag_run'])
    ratio = os.path.getsize(get_full_path(context['logical_date'], 'json')) \
          / os.path.getsize(get_full_path(prev.logical_date, 'json'))
    if ratio > 1.5 or ratio < 0.5:
        raise Exception(f'Skew detected: ratio={ratio:.2f}')
```

### Standard deviation based detection

```sql
-- Day-over-day row count with stddev threshold
WITH daily AS (
  SELECT event_date, COUNT(*) AS row_count
  FROM analytics.events
  WHERE event_date >= CURRENT_DATE - INTERVAL '30 days'
  GROUP BY event_date
),
stats AS (
  SELECT AVG(row_count) AS avg_ct, STDDEV(row_count) AS std_ct
  FROM daily WHERE event_date < CURRENT_DATE
)
SELECT d.event_date, d.row_count, s.avg_ct,
  CASE
    WHEN ABS(d.row_count - s.avg_ct) > 2 * s.std_ct THEN 'ALERT'
    WHEN ABS(d.row_count - s.avg_ct) > 1.5 * s.std_ct THEN 'WARN'
    ELSE 'OK'
  END AS status
FROM daily d CROSS JOIN stats s
WHERE d.event_date = CURRENT_DATE;
```

For partitioned tables, coefficient of variation (stddev / avg) across partitions surfaces distribution skew:

```sql
-- PostgreSQL partitioned tables: skew ratio across partitions
SELECT
  NOW() AS "time",
  (STDDEV(n_live_tup) / AVG(n_live_tup)) * 100 AS skew_ratio
FROM pg_catalog.pg_stat_user_tables
WHERE relname LIKE 'visits_all_range_%';
-- Alert when skew_ratio > threshold (e.g. 30%)
```

### Seasonality handling

- **Weekday vs weekend**: Compare Monday against prior Mondays, not Sunday.
- **Monthly patterns**: End-of-month billing spikes are expected, not skew.
- **Campaigns/launches**: Coordinate with product teams to widen tolerance before known events; narrow it after the event settles.
- **Separate baselines**: Maintain distinct historical windows for seasonal buckets rather than a single rolling average.

### Alert vs warn

Two thresholds: **warn** (1.5 stddev, logs only) and **alert** (2+ stddev, pages on-call). This reduces noise while catching genuine anomalies.

---

## Lag Detector

Measure how far consumers trail producers.

### Core formula

```
lag = last_available_unit - last_processed_unit
```

### Lag units per system type

| System | Lag unit | What to compare |
|--------|----------|-----------------|
| Kafka | Consumer group offset | Latest offset minus committed offset per partition |
| Batch | Partition date gap | Latest available minus latest processed partition |
| CDC | Replication slot lag | `pg_current_wal_lsn() - confirmed_flush_lsn` |
| Streaming | Watermark delay | Processing time minus event-time watermark |
| Delta Lake | Commit version | Latest written version minus last read version |

### Why P90 beats average

Seven partitions with lag: 10, 5, 30, 2, 3, 5, 3 seconds. Average = 8 s (hides the 30 s outlier). P90 = 18 s (surfaces it). Use percentile for overall health, MAX for worst-case partition detection. Combine both in dashboards; do not use average alone.

### Kafka consumer lag monitoring

```yaml
# Prometheus alert rule
- alert: KafkaConsumerLagHigh
  expr: kafka_consumergroup_lag_sum{group="visits_ingest"} > 50000
  for: 5m
  labels: { severity: warning }
  annotations:
    summary: "Consumer group {{ $labels.group }} lag exceeds 50k offsets"
```

```python
# Spark Structured Streaming: push per-partition lag
class LagReporter(StreamingQueryListener):
    def onQueryProgress(self, event):
        latest = self._read_last_available_offsets()
        current = json.loads(event.progress.sources[0].endOffset)['visits']
        for partition, value in current.items():
            lag_gauge.labels(partition=partition).set(latest[int(partition)] - value)
        push_to_gateway('localhost:9091', job='visits_reader_lag',
                        registry=lag_gauge._registry)
```

### Delta Lake batch lag

```python
# Scheduled job with availableNow trigger
visits_stream = spark_session.readStream.table('default.visits')
(visits_stream.writeStream
    .trigger(availableNow=True)
    .option('checkpointLocation', checkpoint_dir)
    .format('console')
    .start()
    .awaitTermination())

# Read last processed version from query progress
last_read = query.lastProgress['sources'][0]['endOffset']['reservoirVersion']
metrics_gauge.set(last_read)

# Read last written version from history
last_written = (spark_session.sql('DESCRIBE HISTORY default.visits')
    .selectExpr('MAX(version)').collect()[0][0])

# Alert when (last_written - last_read) > version threshold
```

### SQL batch lag measurement

```sql
SELECT
  MAX(partition_date)                                              AS latest_available,
  (SELECT MAX(partition_date) FROM etl.processed_partitions
   WHERE dataset = 'visits')                                       AS latest_processed,
  DATEDIFF('hour',
    (SELECT MAX(partition_date) FROM etl.processed_partitions
     WHERE dataset = 'visits'),
    MAX(partition_date))                                           AS lag_hours
FROM raw.visits_partitions;
-- Alert when lag_hours > SLA threshold
```

---

## SLA Miss Detector

Track whether datasets are available by the time consumers need them, not merely whether a pipeline eventually finished.

### Processing time vs event time SLA

Late-arriving data can cause event-time SLA misses even when processing-time SLA is met. These are separate problems:

- **Processing-time SLA**: Was the job fast enough? Measured by `end_time - start_time`.
- **Event-time SLA**: Did data arrive at the consumer on time relative to the event clock? A five-minute network buffer at the producer can break event-time SLA even when the consumer processes instantly.

Monitor both independently when late arrivals are plausible.

### SLA definition template

```yaml
sla_definitions:
  - dataset: visits_mart
    cadence: hourly
    deadline: "HH:45"           # ready 45 min after hour start
    max_processing_time: 30m
    owner: data-engineering
    escalation:
      - tier1: slack-channel-data-alerts    # 0-15 min past deadline
      - tier2: page-on-call-data-eng        # 15-30 min past deadline
      - tier3: page-eng-manager             # 30+ min past deadline
```

### Tracking approach

Record pipeline start/end, compare against deadline:

```sql
CREATE TABLE IF NOT EXISTS observability.sla_tracking (
  dataset      TEXT NOT NULL,
  run_date     DATE NOT NULL,
  started_at   TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  deadline     TIMESTAMPTZ NOT NULL,
  sla_met      BOOLEAN GENERATED ALWAYS AS (completed_at <= deadline) STORED,
  miss_minutes INT GENERATED ALWAYS AS (
    GREATEST(0, EXTRACT(EPOCH FROM completed_at - deadline)::INT / 60)) STORED,
  PRIMARY KEY (dataset, run_date)
);
```

### Airflow SLA definition

```python
@task(sla=datetime.timedelta(seconds=40 * 60))
def process_batch():
    pass

# SLA timer starts from the DAG execution_time, not task start time.
# Account for upstream task delays when setting the threshold.
```

### Spark Structured Streaming SLA tracking

```python
class SlaTracker(StreamingQueryListener):
    def __init__(self, sla_seconds):
        self.sla_seconds = sla_seconds

    def onQueryProgress(self, event):
        duration_ms = event.progress.batchDuration
        if duration_ms and duration_ms > self.sla_seconds * 1000:
            sla_miss_counter.labels(job=event.progress.name).inc()
            push_to_gateway('localhost:9091', job='sla_tracker',
                            registry=sla_miss_counter._registry)
```

### Flink processing-time SLA (record-level)

```python
# Job 1: stamp each record with processing start time
def map_json_to_reduced_visit(json_payload: str) -> str:
    return json.dumps({
        'start_processing_time_unix_ms': time.time_ns() // 1_000_000,
        # ... other fields
    })

# Job 2: aggregate per minute window, emit P95 processing duration
sla_query_datastream \
    .key_by(extract_grouping_key) \
    .window(TumblingEventTimeWindows.of(Time.minutes(1))) \
    .aggregate(PercentilesAggregateFunction())
# Alert when P95 processing_time_ms > SLA threshold
```

### Alert escalation tiers

1. **Tier 1 (0-15 min past deadline)**: Slack notification. Often self-resolves within the window.
2. **Tier 2 (15-30 min)**: Page on-call data engineer. Investigate root cause.
3. **Tier 3 (30+ min)**: Escalate to eng manager, notify impacted consumer teams.

---

## Dataset Tracker (Coarse Lineage)

Maintain a dependency graph: source, transform, mart, dashboard.

```
kafka_raw_visits  (Infra)
    ↓ [Airflow ingest]
visits_staging    (Analytics)
    ↓ [dbt transform]
visits_mart       (Analytics)
    ↓ [ClickHouse sync]
visits_ch         (BI)
    └─ BI Dashboard (Marketing)
```

### Fully managed approach

Use OpenLineage-compatible tools (Airflow, dbt, Spark) to emit lineage events automatically. Store with Marquez.

```json
{
  "eventType": "COMPLETE",
  "job": { "namespace": "airflow", "name": "visits_ingest" },
  "inputs": [{ "namespace": "kafka", "name": "raw_visits" }],
  "outputs": [{ "namespace": "postgres", "name": "visits_staging" }],
  "run": { "runId": "abc-123" }
}
```

Limitation: fully managed solutions only cover supported job types; custom transformations or in-house runners need explicit declarations.

### Manual dependency tracking

When managed tools are overkill, maintain a YAML registry:

```yaml
datasets:
  - name: visits_mart
    sources: [visits_staging, user_dim]
    transform: dbt
    owner: analytics-team
    consumers: [marketing_dashboard, sales_api]
```

### What to track

- Source → transform → mart → dashboard (at minimum)
- Team ownership per dataset
- Update cadence and SLA per dataset
- Consumer count (blast radius indicator)

### Incident response integration

Query the lineage graph during incidents to answer:

1. What broke downstream from this dataset?
2. Which teams need to be notified?
3. What is the blast radius to set recovery priority?

---

## Fine-Grained Tracker (Column-Level Lineage)

### Query plan analysis approach

Parse SQL execution plans to walk column dependencies bottom-up. Given:

```sql
SELECT CONCAT(u.first_name, d.delivery_address) AS user_with_address
FROM users u JOIN addresses d ON u.id = d.user_id
```

Lineage for `user_with_address`: inputs are `users.first_name` and `addresses.delivery_address`; transformation is CONCAT. Supported natively by Databricks Unity Catalog (`system.access.column_lineage`), Azure Purview, and OpenLineage SQL parser.

### Row-level lineage via metadata headers

Embed job metadata in Kafka message headers or audit columns so any row traces to its producing job:

```python
# Spark: attach lineage headers when writing
visits_to_save.withColumn('headers', F.array(
    F.struct(F.lit('job_name').alias('key'),
             F.lit('visits_decorator').alias('value')),
    F.struct(F.lit('job_version').alias('key'),
             F.lit(job_version).alias('value')),
    F.struct(F.lit('batch_version').alias('key'),
             F.lit(str(batch_number)).alias('value'))
))
# Downstream jobs prepend parent_lineage header to maintain the full chain
visits_to_save.withColumn('headers', F.array(
    # ... current job headers, plus:
    F.struct(F.lit('parent_lineage').alias('key'),
             F.to_json(F.col('headers')).cast('binary').alias('value'))
))
```

### When column-level lineage is worth the cost

- **Regulated data** (GDPR, HIPAA): Must prove which source fields flow into PII columns.
- **Finance/executive KPIs**: Revenue dashboards must trace to source of truth.
- **Complex denormalized tables**: 30+ columns where origin is non-obvious to new team members.

### When it is overkill

- Internal tables with few columns and obvious provenance.
- Early-stage projects where schema changes weekly and lineage goes stale faster than it helps.
- UDFs and programmatic mappings are opaque to query-plan parsers; manual docs are more reliable until tooling catches up.

---

## False Positive Mitigation

### Healthy baseline selection

Compare against the last **known-good** run, not the last run. Tag runs as healthy or unhealthy. If run N was skewed, run N+1 compares against N-1. This prevents the fatality loop where every run after a bad run fires a false alert.

### Seasonal adjustment

Maintain separate baselines for:

- Weekday vs weekend
- Month-start vs month-end
- Known campaign or product-launch periods
- Public holidays

Temporarily widen thresholds before known events; restore defaults after the pattern settles (typically one to three days).

### Maintenance window handling

```yaml
maintenance_windows:
  - name: weekly_db_maintenance
    schedule: "SUN 02:00-04:00 UTC"
    suppress_alerts: [flow_interruption, lag]
  - name: quarterly_migration
    start: "2026-04-25T22:00:00Z"
    end: "2026-04-26T06:00:00Z"
    suppress_alerts: [all]
```

### Alert fatigue prevention

- **Deduplicate**: Group alerts sharing a root cause into one incident.
- **Rate limit**: Do not re-fire within a cooldown window (e.g., 30 min).
- **Severity tiers**: Reserve pages for action-required alerts; Slack for warnings.
- **Periodic review**: Audit hit rates monthly. Tune or remove alerts that fire without triggering action.

---

## Alert Threshold Configuration

```yaml
alerts:
  flow_interruption:
    datasets:
      visits_mart: { threshold_minutes: 60, severity: critical }
      user_dim:    { threshold_minutes: 180, severity: warning }
  skew:
    default_stddev_multiplier: 2.0
    warn_stddev_multiplier: 1.5
  lag:
    kafka_groups:
      visits_ingest: { max_offset_lag: 50000, severity: warning }
    batch:
      visits_mart: { max_lag_hours: 2, severity: critical }
  sla:
    visits_mart: { deadline: "HH:45", escalation_minutes: [0, 15, 30] }
```

---

## Pattern Decision Flow

```
Observability Problem?
│
├── Data unavailable / inconsistent?
│   └── Flow Interruption Detector
│       ├── Continuous: max(updated_at) vs threshold
│       ├── Irregular: historical gap baseline, wider window
│       └── Batch: metadata layer → storage layer → data layer (row count)
│
├── Data volume unexpected?
│   └── Skew Detector
│       ├── Window-to-window comparison (compare healthy baseline, not previous run)
│       ├── Stddev-based for partitioned tables
│       └── Seasonality: weekday/weekend, campaigns, monthly buckets
│
├── Consumer falling behind?
│   └── Lag Detector
│       ├── lag = last_available - last_processed
│       ├── Units: Kafka offset, partition date, LSN, watermark, Delta version
│       └── Aggregate with P90/P95, not average; MAX for worst-case
│
├── Pipeline too slow?
│   └── SLA Miss Detector
│       ├── Processing-time SLA: end_time - start_time > threshold
│       └── Event-time SLA: data arrival vs event clock (separate problem)
│
└── Need to trace root cause?
    ├── Which team owns this dataset?
    │   └── Dataset Tracker
    │       └── Fully managed (OpenLineage + Marquez) or manual YAML registry
    └── Which columns/jobs created this field/row?
        └── Fine-Grained Tracker
            ├── Column-level: query plan analysis
            └── Row-level: metadata headers in Kafka / audit columns
```

---

## Practical Rollout Order

**Phase 1 -- Freshness and ownership (weeks 1-2)**

- Emit `last_successful_update` timestamp from every ingest job.
- Expose "last updated" in every consumer-facing interface.
- Assign an owner, cadence, and SLA to each critical dataset.
- Deploy flow interruption detector for the top three datasets by consumer count.
- Milestone: every critical dataset has an owner and a freshness signal.

**Phase 2 -- Volume and skew (weeks 3-4)**

- Add daily row-count tracking per dataset and partition.
- Implement window-to-window comparison with 50% default tolerance.
- Roll out stddev-based skew detection for partitioned tables.
- Establish healthy-baseline tagging on each pipeline run.
- Milestone: skew alerts firing against healthy baselines with fewer than two false positives per week.

**Phase 3 -- Lag and SLA (weeks 5-7)**

- Deploy Kafka consumer lag monitoring via Prometheus alert rules.
- Add batch lag measurement (partition date gap or Delta version gap).
- Formalize SLA definitions using the YAML template for all consumer-facing datasets.
- Build the SLA tracking table and wire up three-tier escalation.
- Milestone: on-call team receives actionable SLA miss pages with clear owner and escalation path.

**Phase 4 -- Lineage (weeks 8-10)**

- Enable OpenLineage in Airflow and dbt (zero-config for supported job types).
- Stand up Marquez for lineage storage and visualization.
- Document the dependency graph for top marts (source → mart → dashboard).
- Add column-level lineage only for regulated or executive-facing datasets.
- Milestone: incident triage can identify blast radius from the lineage graph within five minutes.

**Phase 5 -- Hardening (ongoing)**

- Audit alert hit rates monthly; tune or remove alerts that fire without triggering action.
- Add seasonal baselines and maintenance window suppression.
- Integrate lineage queries into incident response runbooks.
- Review SLA definitions with consumer teams each quarter.

Do not start with a heavyweight observability platform if basic ownership, SLAs, and dataset contracts are still undefined. Nail freshness first, then layer complexity.
