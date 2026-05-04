# Value Patterns

Ten patterns for increasing data value after ingestion through enrichment, decoration, aggregation, sessionization, and ordering.

---

## Enrichment Patterns

### Static Joiner

**Use case:** JOIN raw events with stable reference data — user profiles, course metadata, product dimensions — to add business context that is not captured at event time.

**SCD Type 2 handling:** When reference data changes over time, maintain validity date ranges per row so historical joins return the attribute value that was true at event time.

```sql
-- Dimension join with effective date range (SCD Type 2)
SELECT
    e.event_id,
    e.event_time,
    e.user_id,
    d.user_name,
    d.market,
    d.plan_tier
FROM events e
JOIN user_dim d
    ON d.user_id = e.user_id
   AND e.event_time BETWEEN d.start_date AND d.end_date;
```

**Late-arriving dimensions:** Materialize CDC or API data into an internal snapshot table before joining. Re-run joins against the stable snapshot, not the live API, using `execution_date` as the idempotent marker.

**Trade-off:** Join complexity scales with dimension history depth (more SCD rows = slower join), but query-time API lookups are eliminated entirely. Prefer static joins unless both sides are genuinely streaming.

### Dynamic Joiner

**Use case:** Streaming JOIN where both sides move — for example, CDC user-profile changes and real-time learning events. Static joins miss matches when the two streams arrive with different latencies.

**Buffering and watermark strategy:** Buffer both streams with time boundaries. A GC watermark evicts records from each buffer once they are too old to match. Buffer size equals the maximum latency difference between the two streams.

```python
# Spark Structured Streaming -- stream-stream join with watermarks
visits   = visits_data.withWatermark('event_time',  '10 minutes')
profiles = profiles_data.withWatermark('update_time', '10 minutes')

result = visits.join(profiles, F.expr('''
    visits.user_id = profiles.user_id AND
    profiles.update_time BETWEEN
        visits.event_time - INTERVAL 5 minutes AND
        visits.event_time + INTERVAL 2 minutes
'''), 'left_outer')
```

**Trade-off:** Latency vs completeness. Late data beyond the GC watermark is evicted before a match can be found. Wider watermarks reduce misses at the cost of higher state store memory. Larger buffers increase match completeness but consume proportionally more memory.

---

## Decoration Patterns

### Wrapper

**Use case:** Keep raw fields intact while adding computed fields alongside them. Consumers get enriched output without losing the original values for debugging or audit.

```sql
-- SELECT * keeps raw fields; computed columns are added alongside
SELECT
    *,
    CASE WHEN user.connected_since IS NULL
         THEN false
         ELSE true
    END                                               AS is_connected,
    CONCAT_WS('-', page, referral)                    AS page_key,
    DATEDIFF('minute', session_start, session_end)    AS session_duration_min
FROM raw_events;
```

**When to wrap vs create a new table:** Wrap when consumers need both raw and computed in one scan and the output grain is unchanged. Create a separate table when the computed output has a different grain, a different primary consumer, or needs independent versioning.

**Lineage preservation principle:** Make the boundary between original and derived fields explicit. Options: nest raw fields in a struct (`raw.event_time`), use a naming prefix for computed columns (`calc_duration_min`), or maintain a separate derived table joined by key.

### Metadata Decorator

**Use case:** Track technical metadata — job version, execution time, source file, batch ID — so operations teams have full pipeline visibility without polluting business marts.

**Fields to track:** `job_version`, `execution_time`, `source_file`, `batch_id`, `row_count`, `schema_version`.

**Separate metadata table (recommended):**

```sql
CREATE TABLE pipeline_metadata (
    execution_date TIMESTAMP NOT NULL, job_version VARCHAR(15) NOT NULL,
    source_file VARCHAR(255), batch_id VARCHAR(50), row_count BIGINT,
    PRIMARY KEY (execution_date, job_version)
);

-- Operations view joins via hidden _metadata_id; analysts never see this join
CREATE VIEW sessions_ops AS
SELECT s.*, m.job_version, m.execution_time, m.source_file
FROM learning_sessions s JOIN pipeline_metadata m ON s._metadata_id = m.execution_date;
```

**Native metadata layer:** When the sink supports header fields (e.g., Kafka message headers), write metadata there so the payload stays clean. Note that not every store supports this — Amazon Kinesis does not.

**Rule:** Never surface `job_version`, `batch_id`, or `execution_time` in dashboards or analyst-facing tables. Metadata belongs in operational views and data quality monitors.

---

## Aggregation Patterns

### Distributed Aggregator

**Use case:** Group key is spread across partitions — no co-location guarantee. Requires shuffle so all records for a key land on one reducer (MapReduce: map → shuffle → reduce).

```python
# PySpark distributed aggregation -- triggers a full shuffle
visits.groupBy('user_id', 'device_type') \
      .agg(F.count('*').alias('visit_count'), F.avg('duration').alias('avg_duration')) \
      .write.format('parquet').save('s3://warehouse/fact_visits/')
```

**Data skew and salting:** When one key has disproportionately many records, salt it across multiple reducers, aggregate per salt, then merge.

```sql
-- Step 1: add random salt; aggregate per (key, salt) -- no single reducer overloaded
SELECT
    group_key,
    FLOOR(RAND() * 4)   AS salt,
    COUNT(*)            AS partial_count,
    SUM(amount)         AS partial_sum
FROM large_table
GROUP BY group_key, salt;

-- Step 2: remove salt; combine partial results
SELECT
    group_key,
    SUM(partial_count)  AS total_count,
    SUM(partial_sum)    AS total_sum
FROM salted_partials
GROUP BY group_key;
```

**When shuffle cost is acceptable:** Batch fact-table builds (daily DAU, completion rates) where throughput matters more than latency. Spark Adaptive Query Execution can also auto-detect and handle skew without manual salting.

**When prohibitive:** Real-time or micro-batch pipelines where network exchange dominates end-to-end latency. Consider local aggregation or pre-partitioning instead.

### Local Aggregator

**Use case:** Records for a key are already co-located by partition design. Shuffling is wasted work — aggregate within each partition without any network exchange.

**Requirements:** Static partitioning schema (partition count does not change after setup) and records with the same key always routing to the same partition.

**ClickHouse Materialized Views — canonical example:**

```sql
-- AggregatingMergeTree accumulates partial states locally per shard, no cross-shard shuffle
CREATE MATERIALIZED VIEW fact_dau_mv
ENGINE = AggregatingMergeTree()
ORDER BY (event_date, user_id)
AS
SELECT
    toDate(event_time)  AS event_date,
    user_id,
    countState()        AS event_count,
    sumState(duration)  AS total_duration
FROM learning_events   -- MergeTree table keyed by (user_id, event_time)
GROUP BY event_date, user_id;

-- Query: merge partial states at read time with countMerge / sumMerge
SELECT event_date, user_id,
    countMerge(event_count)   AS events,
    sumMerge(total_duration)  AS duration_sec
FROM fact_dau_mv
GROUP BY event_date, user_id;
```

**Performance advantage:** No network shuffle, sub-second materialization latency on insert, scales linearly with shard count. If a consumer needs a different group key, fall back to distributed aggregation for that query only.

**Limitation:** If partition count changes (re-sharding), the entire dataset must be reprocessed to rebuild co-location guarantees.

---

## Sessionization Patterns

### Incremental Sessionizer

**Use case:** Build session facts from partitioned event data in batch. Sessions may span partition boundaries (e.g., a learning session that starts at 23:50 and ends at 00:15 the next day).

**Three storage areas:**

1. **Input** — raw partitioned events (hourly or daily)
2. **Pending** — sessions still accumulating (not yet finalized)
3. **Completed** — final session facts ready for downstream consumption

**Session building with gap detection (30-minute inactivity):**

```sql
WITH ordered AS (
    SELECT user_id, event_time, page,
           LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_time
    FROM raw_events WHERE event_date = '{{ ds }}'
),
starts AS (
    SELECT *,
        CASE WHEN prev_time IS NULL OR DATEDIFF('minute', prev_time, event_time) > 30
             THEN 1 ELSE 0 END AS is_new_session
    FROM ordered
),
with_seq AS (
    SELECT *,
        SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_seq
    FROM starts
)
SELECT user_id,
    MD5(CONCAT(user_id, '-', session_seq))               AS session_id,
    MIN(event_time)                                       AS session_start,
    MAX(event_time)                                       AS session_end,
    DATEDIFF('minute', MIN(event_time), MAX(event_time)) AS duration_min,
    COUNT(*) AS event_count, COUNT(DISTINCT page)        AS distinct_pages
FROM with_seq
GROUP BY user_id, session_seq;
```

**Cross-partition session continuation:**

```sql
CREATE TEMPORARY TABLE classify AS
SELECT COALESCE(p.session_id, n.session_id)        AS session_id,
       LEAST(p.start_time, n.start_time)            AS start_time,
       GREATEST(p.last_event, n.last_event)         AS last_event,
       ARRAY_CAT(p.pages, n.pages)                  AS pages,
       CASE WHEN n.user_id IS NULL THEN p.expiration
            ELSE '{{ execution_date + 2 days }}' END AS expiration
FROM new_partition_sessions n
FULL OUTER JOIN pending_sessions p ON n.session_id = p.session_id;

INSERT INTO pending_sessions   SELECT * FROM classify WHERE expiration > '{{ execution_date }}';
INSERT INTO completed_sessions SELECT * FROM classify WHERE expiration = '{{ execution_date }}';
```

**Replay dependency:** Sessionization is forward-dependent. Replaying one partition forces replay of all subsequent partitions because pending sessions carry state forward. Use `execution_date` as an idempotent marker — DELETE existing rows for that date before re-inserting. Mark pending sessions with `is_final = false` so consumers know a session may still grow.

### Stateful Sessionizer

**Use case:** Real-time session output where batch latency is unacceptable. Maintains a state store per session key, checkpointing to durable storage.

**Spark Structured Streaming** (`flatMapGroupsWithState` / `applyInPandasWithState`):

```python
sessions = (
    visits.withWatermark('event_time', '1 minute')
          .groupBy('visit_id')
          .applyInPandasWithState(
              func=map_visits_to_session,
              outputStructType=session_schema,
              stateStructType=state_schema,
              outputMode="update",
              timeoutConf="EventTimeTimeout")   # expire tied to watermark progress
)
```

**Apache Flink session windows** (simpler, declarative):

```python
visits.key_by(VisitIdSelector()) \
      .window(EventTimeSessionWindows.with_gap(Time.minutes(10))) \
      .process(VisitToSessionConverter())
```

**State checkpointing:** Checkpoint to durable storage (S3, HDFS) at regular intervals. On restart, resume from the last checkpoint with at-least-once semantics.

**Watermark configuration:** Set to the maximum expected event lateness. Wider watermarks capture more late data but increase state store size and output latency. Always use event-time expiration — processing-time expiration is unpredictable under backpressure.

**Trade-off:** Real-time latency (seconds) vs state store size. State rebalancing on scale-out is slower than stateless processing.

---

## Ordering Patterns

### Bin Pack Orderer

**Use case:** Sink supports bulk writes with partial commit semantics — some records in a batch succeed while others fail. Without ordering guarantees, retries break the event sequence.

**Implementation:** Sort by grouping key + time. Pack sorted records into bins (one key per bin). Deliver bins sequentially — wait for acknowledgment before advancing to the next bin.

```python
events.sortWithinPartitions(['visit_id', 'event_time']) \
      .foreachPartition(write_ordered_bins)

def write_ordered_bins(records):
    bins, last_key, idx = [], None, 0
    for r in records:
        if r.visit_id != last_key:
            idx, last_key = 0, r.visit_id
        if len(bins) <= idx:
            bins.append([])
        bins[idx].append(r)
        idx += 1
    for bin_records in bins:
        sink.put_records(bin_records)  # block until ack
```

**When ordering matters:** Fragile sinks with partial commit APIs (Kinesis, DynamoDB conditional writes) or ordered consumers that depend on strict event sequence (CDC replay, stateful downstream processors).

**Gotcha:** Ordering is guaranteed within one execution only. Full job retry on already-emitted records may break ordering at the sink — pair with an idempotency pattern on the sink side.

### FIFO Orderer

**Use case:** Simple low-volume scenario requiring strict ordering where throughput is not a concern. No buffering or bin-packing overhead.

```python
# Simple FIFO: one record at a time, block until ack
producer = Producer({'max.in.flight.requests.per.connection': 1})
for record in records:
    producer.produce(record)
    producer.flush()

# Kinesis: use SequenceNumberForOrdering to enforce FIFO within a shard
previous_seq = None
for record in records:
    result = kinesis.put_record(StreamName='my-stream', Data=record.to_bytes(),
                                PartitionKey=record.key,
                                SequenceNumberForOrdering=previous_seq)
    previous_seq = result['SequenceNumber']
```

**Trade-off:** One request per record means high I/O overhead and low throughput. Acceptable for low-volume streams or strict single-entity ordering where bin packing complexity is not justified. FIFO ordering does not imply exactly-once delivery — add an idempotency pattern on the consumer side.

---

## Pattern Selection Guide

| Need | Pattern | Complexity |
|------|---------|------------|
| Add reference context | Static Joiner | Low |
| Join two streams | Dynamic Joiner | High |
| Keep raw + computed | Wrapper | Low |
| Track pipeline metadata | Metadata Decorator | Low |
| Aggregate across partitions | Distributed Aggregator | Medium |
| Aggregate co-located keys | Local Aggregator | Low |
| Build sessions in batch | Incremental Sessionizer | Medium |
| Build sessions real-time | Stateful Sessionizer | High |
| Ordered bulk delivery | Bin Pack Orderer | Medium |
| Simple ordered delivery | FIFO Orderer | Low |

---

## Practical Defaults

Sensible starting points for analytics platforms built on ClickHouse with batch session pipelines.

**Static joiner for user and course dimensions:**

```sql
SELECT
    le.event_id,
    le.event_time,
    le.user_id,
    ud.user_name,
    ud.market,
    cd.course_name,
    cd.category
FROM learning_events le
JOIN user_dim ud
    ON ud.user_id = le.user_id
   AND le.event_time BETWEEN ud.start_date AND ud.end_date
JOIN course_dim cd
    ON cd.course_id = le.course_id
   AND le.event_time BETWEEN cd.start_date AND cd.end_date;
```

SCD Type 2 on both dimensions ensures historical backfills resolve the correct attribute values for the event timestamp.

**Local aggregator for ClickHouse materialized views:**

```sql
CREATE MATERIALIZED VIEW fact_dau_mv ENGINE = AggregatingMergeTree()
ORDER BY (event_date, market) AS
SELECT toDate(event_time) AS event_date, market,
       uniqState(user_id) AS unique_users, countState() AS total_events
FROM enriched_learning_events
GROUP BY event_date, market;
```

No cross-shard shuffle when ClickHouse distributes by the same key as the aggregation. Merge at query time with `uniqMerge` / `countMerge`.

**Incremental sessionizer for learning session facts:** Use the 3-area pattern (input, pending, completed) with gap detection. Set the inactivity threshold to your product definition (5 minutes for learning sessions). Mark pending sessions `is_final = false` so consumers know a session may still grow.

**Metadata decorator for pipeline tracking:** Maintain a `pipeline_metadata` table keyed by `(execution_date, job_version)`. Expose via an operational view for debugging. Never surface `batch_id`, `job_version`, or `execution_time` in analyst-facing marts.
