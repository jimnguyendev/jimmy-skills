# Reliability Patterns

Ingestion patterns, idempotency patterns, ordering patterns, and decision guidance for pipelines that are safe to rerun, retry, and backfill.

> **Disclaimer:** Code and config examples are derived from "Data Engineering Design Patterns" by Bartosz Konieczny (Ch2, Ch4). Official documentation for Debezium, Apache Spark, Delta Lake, and Apache Airflow remains authoritative.

---

## Ingestion Patterns

Eight patterns covering the full ingestion spectrum from simple batch to event-driven streaming.

---

### Pattern 1: Full Loader

**Problem:** Need to bring an entire dataset from source into destination with no state tracking.

**Solution:** Copy the complete dataset every run. Use `pg_dump`, `aws s3 sync`, `COPY INTO`, Spark read/write, or infrastructure-level replication (S3 bucket replication).

**Implementation:** No watermark, no bookmark. Each run is a clean replace.

**Consequences:**

- Simplest pattern, zero state to manage.
- Does not scale — dataset grows, job duration grows linearly.
- High network and compute cost even when only a few rows changed.
- Use for: initial loads, small reference tables, bootstrapping before switching to incremental.

---

### Pattern 2: Incremental Loader

**Problem:** Dataset too large to full-load every run. Only new or changed rows are needed.

**Solution:** Two approaches — delta column or time-partitioned.

```sql
-- Approach 1: Delta column
SELECT * FROM source_table
WHERE updated_at > '2024-01-15T00:00:00Z';

-- Backfill-safe: bound the window
SELECT * FROM source_table
WHERE updated_at BETWEEN '2024-01-01' AND '2024-01-15';

-- Approach 2: Time-partitioned (load only the new partition directory)
-- e.g., read s3://bucket/events/date=2024-01-16/
```

**Consequences:**

- Hard deletes are invisible: deleted rows vanish from delta queries. Mitigation: soft deletes or pair with CDC.
- Late data: `event_time` as delta column misses late arrivals. Use `ingestion_time` instead.
- Backfill becomes a bounded full load with an explicit date range.
- Insert-only tables (ClickHouse `ReplacingMergeTree`) sidestep update issues by deduplicating on read.

---

### Pattern 3: Change Data Capture (CDC)

**Problem:** Incremental loader is too slow for sub-minute latency, and you need to capture DELETEs without soft-delete workarounds.

**Solution:** Read directly from the database commit log (WAL in PostgreSQL). Every INSERT/UPDATE/DELETE streams as an event with operation metadata.

```
PostgreSQL (WAL) → Debezium → Kafka Connect → Kafka topics → Consumers
```

**Debezium PostgreSQL connector config:**

```json
{
  "name": "visits-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.dbname": "postgres",
    "schema.include.list": "app_schema",
    "topic.prefix": "cdc",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "dbz_publication"
  }
}
```

Each table `app_schema.events` produces Kafka topic `cdc.app_schema.events`. Each CDC payload includes operation type (`c`/`u`/`d`), modification timestamp, and before/after column values.

**Delta Lake Change Data Feed (lighter alternative for lakehouse):**

```sql
ALTER TABLE events SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
-- Each row includes: _change_type, _commit_version, _commit_timestamp
SELECT * FROM table_changes('events', 2, 5);
```

**Consequences:**

- Sub-minute latency; captures deletes natively.
- Complex setup: WAL replication config, Kafka cluster, Debezium connector permissions.
- Only captures changes after CDC start — combine with Full Loader for historical data.
- Stream joins have temporal mismatches (stream A may lag stream B).

---

### Pattern 4: Passthrough Replicator

**Problem:** Copy data between environments (prod → staging) preserving the exact original format — no transformation.

**Solution:** EL job (Extract-Load, no Transform). Raw byte copy, infrastructure replication (S3 bucket replication), or Kafka MirrorMaker.

**Key principle:** Use raw text/byte APIs instead of typed parsers to avoid silent transformations (date format conversion, float rounding).

**Consequences:**

- Use push from production rather than pull from lower environments (security boundary).
- Must replicate metadata too: Parquet footers, Delta Lake commit log, Kafka headers and partition order.
- If PII is present, use Transformation Replicator instead.

---

### Pattern 5: Transformation Replicator

**Problem:** Replicate data that contains PII which cannot leave the production boundary.

**Solution:** Apply transformation between read and write to remove or anonymize sensitive fields.

```sql
-- Option 1: Column reduction
SELECT * EXCEPT (ip, latitude, longitude) FROM visits;

-- Option 2: Column-level grants
GRANT SELECT (visit_id, event_time) ON TABLE visits TO readonly_user;

-- Option 3: Column-level masking
SELECT visit_id, event_time, SHA256(email) AS email_hash,
       SUBSTRING(full_name, 2) AS name_partial
FROM visits;
```

**Consequences:**

- PII field definitions drift over time — use data contracts or catalog tags to auto-detect sensitive fields.
- Text formats (JSON, CSV) risk silent schema errors; prefer typed formats (Parquet, Avro) for replication.

---

### Pattern 6: Compactor

**Problem:** Real-time ingestion produces thousands of small files. Metadata overhead (listing files) can dominate execution time — 70% listing, 30% actual processing.

**Solution:** Merge small files into fewer large files periodically.

| Technology | Command |
|---|---|
| Delta Lake | `DeltaTable.forPath(spark, path).optimize().executeCompaction()` then `vacuum()` |
| Apache Iceberg | `rewrite_data_files` action |
| Apache Hudi | Merge-on-Read compaction |
| Kafka | `log.cleanup.policy=compact` (keeps latest value per key) |
| PostgreSQL | `VACUUM` (reclaim dead tuples) |
| ClickHouse | Automatic background merge; force with `OPTIMIZE TABLE ... FINAL` |

**Consequences:**

- Compaction burns compute — tune frequency (too often = cost, too rare = consumer overhead).
- Raw files (JSON/CSV) lack ACID; run compaction on open table formats for safety.
- Old files persist after compaction — run VACUUM to reclaim disk.

---

### Pattern 7: Readiness Marker

**Problem:** Downstream consumers start reading before upstream finishes writing, producing incomplete results.

**Solution:** Emit an explicit signal that data is complete before consumers are allowed to proceed.

```python
# Approach 1: Flag file — Airflow sensor waits for _SUCCESS
from airflow.sensors.filesystem import FileSensor

wait_for_data = FileSensor(
    task_id='wait_for_upstream',
    filepath=f'{input_path}/_SUCCESS',
    mode='reschedule'
)

# Approach 2: Partition convention
# Job running for partition N implies partition N-1 is complete.
# e.g., job writing date=2024-01-11 signals date=2024-01-10 is ready.
```

**Consequences:**

- No enforcement — consumers can bypass the marker and read incomplete data.
- Late data complicates partition markers: if partition closes before late events arrive, define whether the partition is mutable or immutable.

---

### Pattern 8: External Trigger

**Problem:** Data arrives unpredictably. Fixed-interval polling wastes compute when no data is available.

**Solution:** Push-based trigger — producer notifies the pipeline when new data is ready.

```python
# Airflow DAG with no schedule (triggered externally only)
with DAG('devices-loader', schedule_interval=None):
    load_task = ...

# AWS Lambda fires on S3 object creation, triggers Airflow DAG run
def lambda_handler(event, ctx):
    payload = {
        'conf': {
            'file_to_load': event['Records'][0]['s3']['object']['key'],
            'dag_run_id': f'External-{ctx.aws_request_id}'
        }
    }
    requests.post(
        f'{airflow_url}/dags/devices-loader/dagRuns',
        data=json.dumps(payload),
        headers={'Content-Type': 'application/json'}
    )
```

**Consequences:**

- Saves compute — runs only when data exists.
- Enrich trigger payloads with metadata (version, event time, envelope) to support debugging.
- Trigger failure = lost event = lost data. Pair with a Dead-Letter pattern.

---

## Idempotency Patterns

Seven patterns in three families ensuring reruns produce identical results.

---

## Family 1: Overwrite (Each Run Produces Full State)

Full-state pipelines output the complete dataset on every run. Making them idempotent means replacing the previous output entirely and atomically.

---

### Pattern 1: Fast Metadata Cleaner

**Problem:** DELETE-then-INSERT degrades on large partitions. The schema scan and data rewrite make daily jobs progressively slower. After weeks of growth, DELETE performance degrades significantly.

**Solution:** Replace DELETE with metadata-only operations. TRUNCATE and DROP operate at the metadata layer — they skip the data rewrite.

```sql
-- TRUNCATE: metadata-only, instant on any size
TRUNCATE TABLE visits_2024_w03;

-- DROP + CREATE: also metadata-only
DROP TABLE IF EXISTS visits_2024_w03;
CREATE TABLE visits_2024_w03 (LIKE visits INCLUDING ALL);
```

Partition into weekly (or daily) tables. Expose the full dataset through a VIEW that unions all partitions.

**Airflow orchestration — branch on partition boundary:**

```python
from airflow.operators.python import BranchPythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator

def retrieve_path_for_table_creation(**context):
    ex_date = context['execution_date']
    should_create = ex_date.day_of_week == 1 or ex_date.day_of_year == 1
    return 'create_weekly_table' if should_create else 'skip'

check_boundary = BranchPythonOperator(
    task_id='check_if_monday',
    python_callable=retrieve_path_for_table_creation
)
create_weekly = PostgresOperator(
    task_id='create_weekly_table',
    sql='CREATE TABLE visits_w{{ execution_date.isocalendar()[1] }} (LIKE visits INCLUDING ALL)'
)
insert_data = PostgresOperator(
    task_id='insert_data',
    sql='INSERT INTO visits_w{{ execution_date.isocalendar()[1] }} SELECT * FROM staging_visits'
)
refresh_view = PostgresOperator(
    task_id='refresh_view',
    sql='CREATE OR REPLACE VIEW visits AS SELECT * FROM visits_w{{ execution_date.isocalendar()[1] }}'
)

check_boundary >> [create_weekly >> insert_data, insert_data] >> refresh_view
```

**Consequences:**

- Coarse backfill granularity: replaying one day in a weekly table requires rerunning the whole week.
- Cannot backfill a single entity (user, device) — always operates on the whole partition.
- Metadata limits apply: BigQuery 4K partition limit, Redshift 200K table limit — many pipelines exhaust quotas.
- Optional field additions are free; adding required fields triggers reprocessing.

---

### Pattern 2: Data Overwrite

**Problem:** Object stores (S3, GCS) lack TRUNCATE/DROP. Need idempotency without metadata operations.

**Solution:** Use data-layer replacement commands that atomically replace partition content.

```sql
-- Option A: DELETE + INSERT (two statements — wrap in a transaction)
DELETE FROM devices WHERE load_date = '2024-01-15';
INSERT INTO devices SELECT * FROM devices_staging WHERE load_date = '2024-01-15';

-- Option B: INSERT OVERWRITE (atomic single statement)
INSERT OVERWRITE INTO devices PARTITION (load_date = '2024-01-15')
SELECT * FROM devices_staging WHERE state = 'valid';

-- Option C: BigQuery bq load with replace
-- bq load dedp.devices gs://devices/in_20240101.csv ./schema.json --replace=true
```

```python
# Spark / Delta Lake: conditional partition overwrite
df.write \
    .mode('overwrite') \
    .option('replaceWhere', "load_date = '2024-01-15'") \
    .format('delta') \
    .save(path)
```

**Consequences:**

- Always partition first — unpartitioned overwrite rewrites the entire table on every run.
- DELETE does not reclaim disk immediately — schedule VACUUM.
- Time-travel stores (BigQuery, Snowflake, Delta Lake) keep old versions until retention expires.

---

## Family 2: Merge (Each Run Contains Only Changes)

Incremental pipelines deliver only the delta — new inserts, updates, and deletes. Making them idempotent requires merging against the existing state without duplicating or losing records.

---

### Pattern 3: Merger

**Problem:** Incremental changes (inserts, updates, deletes) sourced from CDC or delta sync must be applied without creating duplicates.

**Solution:** MERGE/UPSERT with explicit three-way handling for insert, update, and delete cases.

```sql
MERGE INTO devices_target AS target
USING devices_input AS input
  ON target.device_id = input.device_id
 AND target.version   = input.version
WHEN MATCHED AND input.is_deleted = true THEN
  DELETE
WHEN MATCHED AND input.is_deleted = false THEN
  UPDATE SET full_name  = input.full_name,
             updated_at = input.updated_at
WHEN NOT MATCHED AND input.is_deleted = false THEN
  INSERT (device_id, full_name, version, updated_at)
  VALUES (input.device_id, input.full_name, input.version, input.updated_at);
```

**Critical guard:** The NOT MATCHED clause must check `is_deleted = false`. Without it, tombstone rows get inserted and can never be removed on a future run.

**dbt incremental model example:**

```sql
-- models/fact_sessions.sql
{{ config(
    materialized='incremental',
    unique_key='session_id',
    merge_update_columns=['pages', 'updated_at'],
) }}

SELECT
    session_id,
    user_id,
    pages,
    FALSE AS is_deleted,
    CURRENT_TIMESTAMP AS updated_at
FROM {{ ref('stg_sessions') }}
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Consequences:**

- Requires an immutable join key. If the key changes, merge inserts a duplicate instead of updating.
- I/O intensive: operates at data block level. Modern engines optimize with metadata skipping.
- Backfill hazard: replaying an old run against a table already containing future data produces a hybrid state that never existed. Use Stateful Merger to solve this.

---

### Pattern 4: Stateful Merger

**Problem:** Plain Merger produces inconsistent results during backfills because the target already contains data from future runs.

**Solution:** Track table versions per execution in a state table. Before merging a replayed run, restore the table to the version that existed just before that run originally executed.

**State table:**

```sql
CREATE TABLE IF NOT EXISTS pipeline_versions (
    execution_time    TIMESTAMP NOT NULL,
    delta_table_version INT NOT NULL
);
```

**Backfill detection and restore logic (PySpark / Delta Lake):**

```python
# Step 1: Detect backfill
last_merge_version = (
    spark.sql('DESCRIBE HISTORY default.devices')
         .filter("operation = 'MERGE'")
         .selectExpr('MAX(version) AS v')
         .collect()[0].v
)
maybe_prev = spark.sql(
    f"SELECT delta_table_version FROM pipeline_versions "
    f"WHERE execution_time = '{prev_execution_time}'"
).collect()

# Step 2: Restore or truncate
if not maybe_prev:
    # First run ever — start clean
    spark.sql('TRUNCATE TABLE default.devices')
elif maybe_prev[0].delta_table_version < last_merge_version:
    # Backfill detected: future merges have already run — restore
    DeltaTable.forName(spark, 'devices').restoreToVersion(
        maybe_prev[0].delta_table_version
    )
# else: normal sequential run — no restore needed

# Step 3: Standard MERGE (see Merger pattern)
# Step 4: Record new version
new_version = spark.sql(
    'SELECT MAX(version) FROM (DESCRIBE HISTORY default.devices)'
).collect()[0][0]
spark.sql(
    f"INSERT INTO pipeline_versions VALUES ('{execution_time}', {new_version})"
)
```

**Airflow DAG branching for backfill detection:**

```python
# Pseudocode — branch before the merge task
def detect_backfill(**context):
    prev_version = get_recorded_version(context['prev_execution_date'])
    current_version = get_current_table_version()
    if prev_version is None:
        return 'truncate_table'
    elif prev_version < current_version:
        return 'restore_table_version'
    else:
        return 'skip_restore'

branch = BranchPythonOperator(task_id='detect_backfill', python_callable=detect_backfill)
restore = SparkSubmitOperator(task_id='restore_table_version', ...)
truncate = PostgresOperator(task_id='truncate_table', sql='TRUNCATE TABLE devices')
skip = DummyOperator(task_id='skip_restore')
merge = SparkSubmitOperator(task_id='merge_changes', ...)
record = PostgresOperator(task_id='record_version', sql='INSERT INTO pipeline_versions ...')

branch >> [restore, truncate, skip] >> merge >> record
```

**State table example timeline:**

```
Execution Time | Table Version
2024-10-05     | 1
2024-10-06     | 2
2024-10-07     | 3   ← created during backfill (detected prev=2, last=3)
2024-10-08     | 4
```

**Consequences:**

- Requires a versioned store (Delta Lake, Iceberg) or manual raw-data tracking with `execution_time` column.
- VACUUM retention limits restore depth — if retention is 7 days, backfills beyond 7 days are blocked.
- Compaction creates new versions: use `current_version - 1` instead of relying on the recorded version directly, to skip compaction-generated entries.

---

## Family 3: Database Layer (Keys and Transactions)

Push idempotency responsibility into the database itself rather than managing it in pipeline orchestration.

---

### Pattern 5: Keyed Idempotency

**Problem:** Streaming pipeline retries must not create duplicate sessions or events.

**Solution:** Derive keys exclusively from immutable attributes so retries regenerate the same key and the database enforces uniqueness.

```python
# WRONG: event_time is mutable — late data shifts the minimum, changing the key on retry
session_id = hash(user_id, min_event_time)   # ❌

# CORRECT: append_time (Kafka record timestamp) is immutable and monotonic
session_id = hash(user_id, min_append_time)  # ✓
```

**Spark Structured Streaming — extract append time from Kafka:**

```python
from pyspark.sql import functions as F

(input_data
    .selectExpr('CAST(value AS STRING)', 'timestamp')
    .select(F.from_json(F.col('value'), schema).alias('visit'), F.col('timestamp'))
    .selectExpr('visit.*', 'UNIX_TIMESTAMP(timestamp) AS append_time')
    .withWatermark('event_time', '10 seconds')
    .groupBy('user_id')
    .applyInPandas(generate_session_id, output_schema)
    .writeStream
    .trigger(processingTime='1 minute')   # Spark streaming trigger
    .format('scylladb')
    .start()
)

# Kinesis adapter: use approximateArrivalTimestamp instead
spark_session.readStream.format('kinesis').load() \
    .selectExpr('CAST(data AS STRING)', 'approximateArrivalTimestamp AS append_time')
```

**ScyllaDB / Cassandra table (key uniqueness built in):**

```sql
CREATE TABLE sessions (
    session_id  BIGINT,
    user_id     BIGINT,
    pages       LIST<TEXT>,
    ingestion_time TIMESTAMP,
    PRIMARY KEY (session_id, user_id)
);
```

**File/partition naming (also keyed idempotency):**

```python
# Execution time is immutable — replaying the same run always writes the same filename
output_file = f"s3://bucket/visits/{execution_date}.parquet"
# Retry overwrites the same file → idempotent
```

**Consequences:**

- NoSQL (Cassandra, ScyllaDB, HBase): key uniqueness is built in.
- Relational DBs: key conflict raises an error — combine with Merger pattern (adds complexity).
- Kafka deduplicates via async compaction — duplicates are temporarily visible between compaction cycles.
- If early events are deleted by compaction before the key is computed, the key cannot be regenerated — accept that a changed record shape produces a changed key.

---

### Pattern 6: Transactional Writer

**Problem:** Node eviction mid-write causes partial data visible to consumers before the job completes.

**Solution:** Wrap all writes in a database transaction. Changes are invisible until COMMIT; ROLLBACK discards everything.

```sql
-- All-or-nothing semantics
BEGIN;
DELETE FROM devices WHERE load_date = '2024-01-15';
INSERT INTO devices SELECT * FROM staging_devices WHERE load_date = '2024-01-15';
COMMIT;
-- If error at any point: ROLLBACK discards both DELETE and INSERT
```

**Flink + Kafka exactly-once (distributed transactional writer):**

```python
from pyflink.datastream.connectors.kafka import KafkaSink, DeliveryGuarantee

kafka_sink = (KafkaSink.builder()
    .set_bootstrap_servers('localhost:9094')
    .set_delivery_guarantee(DeliveryGuarantee.EXACTLY_ONCE)
    .set_property('transaction.timeout.ms', str(60 * 1000))
    .build())
```

**Consequences:**

- Commit coordination adds latency — consumers see data only after the slowest task commits.
- Kafka + Flink supports transactions; Kafka + Spark does not.
- Idempotency scope is per-transaction only. Backfill = new transaction = same records inserted again. Combine with Merger to deduplicate on replay.

---

### Pattern 7: Proxy (Immutable Dataset)

**Problem:** Audit or legal requirements demand that all historical dataset versions be preserved, while consumers should always see only the latest version.

**Solution:** Write each run to a new immutable internal table. Expose the latest version through a VIEW or manifest.

```sql
-- Run 1: create internal table, point VIEW at it
CREATE TABLE devices_internal_20240115_120000 AS SELECT * FROM staging_devices;
REVOKE INSERT, UPDATE, DELETE ON devices_internal_20240115_120000 FROM pipeline_user;

CREATE OR REPLACE VIEW devices AS
    SELECT * FROM devices_internal_20240115_120000;

-- Run 2: new table, update VIEW (no data deleted)
CREATE TABLE devices_internal_20240116_120000 AS SELECT * FROM staging_devices;
CREATE OR REPLACE VIEW devices AS
    SELECT * FROM devices_internal_20240116_120000;
```

**Airflow: generate unique internal table name per run:**

```python
def get_devices_table_name(**context):
    dag_run = context['dag_run']
    suffix = dag_run.start_date.strftime('%Y%m%d_%H%M%S')
    return f'dedp.devices_internal_{suffix}'
```

**Delta Lake: overwrite is still idempotent, time-travel keeps history:**

```python
df.write.mode('overwrite').format('delta').save(path)
# Restore to any previous version
spark.sql('RESTORE TABLE devices TO VERSION AS OF 3')
```

**Consequences:**

- Not all databases support VIEWs — fall back to manifest files pointing to immutable object-store paths.
- Enforce immutability via WORM policies (S3 Object Lock, Azure immutability) or revoked write permissions.
- Storage grows with every run — set a retention policy for old internal tables.

---

## Ordering Patterns

### Bin Pack Orderer

**Problem:** Partial batch failure reorders records on retry, breaking downstream assumptions about ordering.

**Solution:** Use partial-commit APIs that acknowledge individual records. Track committed offsets per partition so a retry resumes from the exact failure point without reordering.

**Consequences:** Complex client logic. Necessary when ordering has business meaning — financial transactions, event sourcing, audit logs.

---

### FIFO Orderer

**Problem:** Need strict ordering with minimal implementation overhead.

**Solution:** Process records sequentially, one at a time.

**Consequences:** Throughput is bounded by single-record processing rate. Use only for low-volume, order-critical streams where simplicity outweighs throughput.

---

## Backfilling Strategies

### Why Plain Merge Is Not Enough

During a backfill, the target table already contains data produced by future runs. Merging old delta input on top of that state produces a hybrid that never existed in production. Consumers querying during the backfill see inconsistent data.

### State Restore + Version-Aware Replay

1. Look up the table version recorded before the replayed run (from `pipeline_versions`).
2. Restore to that version (Delta `restoreToVersion`, Iceberg snapshot rollback).
3. Merge the original delta input against the restored state.
4. Record the new post-merge version and advance to the next run sequentially.

### Partition-Level vs Full Replay

| Strategy | When to use | Trade-off |
|---|---|---|
| Partition-level replay | Bounded time range, stateless transforms, no cross-partition keys | Fast; misses cross-partition dependencies |
| Full replay | Schema change, key migration, logic change, cross-partition aggregates | Slow; guarantees full consistency |

---

## Anti-Patterns

**Soft deletes without CDC awareness.** Hard-deleted rows vanish from incremental delta queries. Mixing soft and hard deletes without a CDC layer produces silent data loss for downstream consumers.

**Event time as idempotency key with late data.** `min(event_time)` shifts when late events arrive, changing the derived key on retry and creating duplicates in the target. Use `append_time` (monotonic, immutable) for key derivation.

**Merge without state tracking during historical replays.** Replaying past runs without first restoring the table to its pre-run state contaminates results with future data already written. Always use Stateful Merger, or scope the overwrite to the exact partition being replayed.

---

## Decision Tables

### Ingestion mode selection

| Signal | Recommended pattern |
|---|---|
| First load or small reference table | Full Loader |
| Routine batch, large dataset | Incremental Loader |
| Sub-minute latency or delete capture from RDBMS | CDC (Debezium, Delta Lake CDF) |
| Cross-environment copy, no PII | Passthrough Replicator |
| Cross-environment copy, PII present | Transformation Replicator |
| Many small files from streaming ingestion | Compactor |
| Downstream must not read incomplete data | Readiness Marker |
| Data arrives unpredictably | External Trigger |

### Idempotency pattern selection

| Pipeline mode | Failure risk | Write semantics | Recommended pattern |
|---|---|---|---|
| Full refresh, partitioned target, metadata ops available | Low | Overwrite partition | Fast Metadata Cleaner |
| Full refresh, object store or no metadata ops | Low | Overwrite partition | Data Overwrite |
| Incremental, immutable key, no backfill risk | Medium | Merge | Merger |
| Incremental, backfill consistency required | High | Merge + restore | Stateful Merger |
| Streaming, key-value target | Medium | Upsert by key | Keyed Idempotency |
| Multi-statement warehouse load | Medium | Transaction | Transactional Writer |
| Audit trail or legal requirement | Low | Append-only + view | Proxy |

### Quick reference

| Question | Answer |
|---|---|
| Simplest safe daily batch? | Overwrite partitions (Data Overwrite or Fast Metadata Cleaner) |
| Incremental sync with deletes? | Merger with `is_deleted` guard in NOT MATCHED clause |
| Replay-safe incremental sync? | Stateful Merger with version restore |
| Deterministic streaming writes? | Keyed Idempotency from append time |
| No partial writes visible to consumers? | Transactional Writer |
| Immutable audit history with latest-only view? | Proxy pattern |
| Sub-minute latency from RDBMS? | CDC (Debezium) |
| Event-driven, no fixed schedule? | External Trigger |
| Too many small files from streaming? | Compactor (Delta optimize, Iceberg rewrite_data_files) |
