# Tooling Playbook

This reference synthesizes the public Datacamping materials from Week 2 through Week 8 into one practical learning path for a modern data stack.

Official docs for Airflow, Snowflake, dbt, Spark, Kafka, and Deequ remain authoritative. Use this document for practical defaults, examples, and cross-tool handoffs.

For in-repo Datacamping source material, read [sources/README.md](sources/README.md).

## Workflow map

| Stage | Tooling emphasized in Datacamping | Concrete example from the materials |
|---|---|---|
| Orchestration | Airflow | `data_ingestion_postgres_dag` loads web logs, creates a table, inserts rows, checks duplicates, and creates a view |
| Warehouse | Snowflake | `sf_tuts` database, `sf_tuts_wh` warehouse, staged CSV upload, then `COPY INTO` and query checks |
| Transformation | dbt | `booking_dbt` project with seeds, tests, transform models, and analysis models |
| Batch processing | Spark | Taxi parquet data, RDD `map` and `reduceByKey`, Snowflake Spark connector |
| Stream processing | Kafka | `streamPayment` topic, partition sizing, late-data handling with event-time concepts |
| Data quality | Python, SQL, dbt, Deequ | Email validation, `data_quality_results`, `dbt test`, and `VerificationSuite` |
| Automation | Containerization, CI/CD, idempotent reruns | Slim dbt CI/CD and replay-safe pipelines as the main public guidance |

## 1. Orchestrate ingestion with Airflow

### What the public pages emphasize

- Airflow is introduced as a workflow orchestrator, but concept depth is intentionally deferred to the official Airflow docs.
- The practical focus is local setup with Docker and a simple ingestion DAG.
- Two setup styles are shown:
  - the official Docker quick start
  - a custom no-frills stack using `LocalExecutor`

### Concrete examples

- The official path downloads `docker-compose.yaml`, runs `docker-compose up airflow-init`, then starts the full stack with `docker-compose up`.
- The lightweight path strips the stack down to `postgres`, `scheduler`, and `webserver`, removes Celery-related services, and sets `AIRFLOW__CORE__EXECUTOR=LocalExecutor`.
- The custom image pattern uses `apache/airflow:2.9.1`, installs dependencies from `requirements.txt`, and mounts `dags`, `logs`, `plugins`, and `scripts`.
- The bootstrap script initializes the Airflow database, creates an admin user, and starts the webserver.
- The teaching DAG is `data_ingestion_postgres_dag` with a clear chain:
  - `ingest_data`
  - `create_table`
  - `insert_data`
  - `check_duplicate`
  - `create_view`
- The example verifies output with SQL, showing `web_logs` moving from `0` rows before the run to `9` rows after the run.

### Guidance for pack usage

- Prefer the no-frills setup for learning, debugging, and laptop-friendly practice.
- Move to the official stack or managed orchestration only when you need more production-like behavior.
- Keep custom dependencies in the image, not in manually modified running containers.
- Add one post-load SQL check even for toy DAGs.

## 2. Load and query in Snowflake

### What the public pages emphasize

- Snowflake is taught through hands-on warehouse setup rather than architecture theory.
- The public material covers core primitives:
  - connecting with SnowSQL
  - creating a database
  - creating a virtual warehouse
  - staging files
  - loading data into tables
  - running sanity queries

### Concrete examples

- Connect with SnowSQL and switch to a warehouse context.
- Create the demo database:

```sql
CREATE OR REPLACE DATABASE sf_tuts;
```

- Create a small, cost-aware warehouse:

```sql
CREATE OR REPLACE WAREHOUSE sf_tuts_wh WITH
    WAREHOUSE_SIZE='X-SMALL'
    AUTO_SUSPEND = 180
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED=TRUE;
```

- Stage employee CSV files with `PUT file://...employees0*.csv @sf_tuts.public.%emp_basic;`.
- The page distinguishes internal stages from external stages in S3, GCS, or Azure.
- The flow ends with load and query validation, then clean teardown using `DROP DATABASE IF EXISTS` and `DROP WAREHOUSE IF EXISTS`.

### Guidance for pack usage

- Teach Snowflake through a minimal path: create warehouse, stage files, load, query, validate, tear down.
- Default to small warehouses with auto suspend and auto resume for training or low-risk use cases.
- Always pair file loading with post-load checks such as row count, key distinctness, and freshness.
- Use Snowflake examples to explain the warehouse boundary, not as a replacement for upstream orchestration or downstream transformation guidance.

## 3. Transform and test with dbt

### What the public pages emphasize

- The dbt material is project-oriented and grounded in a small working example.
- It teaches the project skeleton, core configuration files, standard commands, and test-backed transformation flow.

### Concrete examples

- The `booking_dbt` project includes `models`, `macros`, `seeds`, `snapshots`, `tests`, and `analyses`.
- `dbt_project.yml` separates `transform` and `analysis` schemas and sets both to `materialized: view`.
- The `.dbt` profile targets Snowflake with `database: warehouse`, `warehouse: compute_dw`, and `threads: 1`.
- The run flow is explicit:
  - `dbt init`
  - `dbt deps`
  - `dbt build`
  - `dbt test`
- The example output shows:
  - seed files `bookings_1`, `bookings_2`, `customers`
  - tests such as `bookings_1_test`, `not_null_customer_id`, and `unique_customer_id`
  - transform models such as `transform.customer`, `transform.combined_bookings`, and `transform.prepped_data`
  - analysis models such as `analysis.hotel_count_by_day` and `analysis.thirty_day_avg_cost`

### Guidance for pack usage

- Treat dbt as the default transformation and publish gate layer once data is in the warehouse.
- Keep project shape explicit: models, tests, macros, seeds, and analyses should each have a clear purpose.
- Use `dbt build` for end-to-end runs and `dbt test` to prove publish readiness.
- Keep examples small enough that users can read the dependency graph in one sitting.

## 4. Run batch processing with Spark

### What the public pages emphasize

- Week 5 is a practical Spark overview rather than a deep internals course.
- The material covers Spark SQL, DataFrames, RDDs, cluster anatomy, and deployment basics.
- It also introduces performance instincts such as reducing shuffle and controlling partitions.

### Concrete examples

- Spark shell startup is shown directly, including the local Web UI.
- The batch example reads parquet taxi data, selects `lpep_pickup_datetime`, `PULocationID`, and `total_amount`, converts to RDD, filters out `PULocationID == 74`, and inspects sample rows.
- `inferSchema` is demonstrated for CSV loading, which is useful for learning but should not be the production default.
- A simple aggregation is implemented with `map` and `reduceByKey` to sum `total_amount` by `PULocationID`.
- The Snowflake Spark connector example runs a count query against `WAREHOUSE.PUBLIC.WEB_EVENTS`.

### Guidance for pack usage

- Prefer DataFrame and Spark SQL APIs by default, and use RDDs when you need lower-level transformations or teaching value.
- Use Spark only when parallel batch processing is warranted by data size or transformation cost.
- Production defaults should include explicit schema management, partition strategy, and shuffle minimization.
- Warehouse integration should be treated as a connector boundary with clear read and write validation.

## 5. Run stream processing with Kafka

### What the public pages emphasize

- Week 6 introduces the Kafka ecosystem through core building blocks:
  - producers
  - consumers
  - brokers
  - topics
  - partitions
  - schema management
  - late data handling
- The public material leans more on concepts and operating ideas than on long code walkthroughs.

### Concrete examples

- Start the stack with `docker-compose up`.
- Confirm the Kafka version with `kafka-topics --version`.
- Create a topic:

```bash
kafka-topics --create --topic streamPayment --bootstrap-server localhost:9092 --partitions 2
```

- Verify topic creation with `kafka-topics --list --bootstrap-server localhost:9092 | grep Payment`.
- The material also discusses late-arriving data, event-time windowing, watermarks, and the effect of partition count on consumer parallelism.

### Guidance for pack usage

- Start stream-processing guidance with event-time semantics and partitioning, not with framework branding.
- Make topic count, consumer count, and late-data policy explicit.
- Use Kafka when latency and event delivery requirements justify the operating overhead.
- Keep a handoff rule between batch and streaming so teams do not adopt Kafka only because it feels more modern.

## 6. Make data quality a publish gate

### What the public pages emphasize

- Week 7 is the most implementation-heavy public section after Airflow and dbt.
- It covers quality dimensions, validation, cleaning, monitoring, audit schema design, and implementation patterns across multiple layers.

### Concrete examples

- Python validation checks a transaction for `transaction_id`, positive `amount`, and a valid email shape.
- A `pandas` cleanup example drops missing transaction IDs, nulls out negative amounts, fills amounts with the median, and sanitizes invalid emails.
- SQL metrics compute completeness, accuracy, and timeliness.
- Snowflake quality tracking stores results in `data_quality_results`.
- A stored procedure `run_data_quality_checks()` is used to centralize checks.
- dbt examples show `not_null` and `accepted_values`, then run `dbt test`.
- Deequ examples use:
  - `AnalysisRunner`
  - `VerificationSuite`
  - `ConstraintSuggestionRunner`

### Guidance for pack usage

- Use a layered quality model:
  - Python or ingestion checks for basic shape and field validity
  - warehouse SQL checks for completeness and business-rule metrics
  - dbt tests for publish gates
  - Deequ for richer profiling or rule discovery
- Store quality results in explicit audit tables instead of burying them only in logs.
- Tie each rule to a consumer-facing failure mode, not to a generic score.
- If the pipeline is rerunnable, quality checks must also be rerunnable and comparable across runs.

## 7. Automate delivery safely

### What the public pages emphasize

- The Week 8 public page is intentionally high level.
- It ties earlier weeks together through DataOps thinking:
  - containerize the stack
  - automate delivery
  - reduce manual error
  - make reruns safe
- It specifically points readers toward a slim dbt CI/CD process and idempotent replay behavior.

### Guidance for pack usage

- Treat the public Week 8 page as principles, not as a full runbook.
- Use it to reinforce:
  - container-first local development
  - CI/CD for dbt or pipeline changes
  - idempotent reruns and replay-safe lakehouse or warehouse flows
  - a clear divide between orchestration success and delivery correctness
- Cross-reference deeper skills when users need detailed reliability, observability, or quality design.

## Practical defaults across the stack

- Start with batch unless the consumer need is genuinely real time.
- Keep local learning stacks lighter than production-like stacks.
- Prove correctness with SQL, tests, or consumer-facing checks after every stage.
- Make idempotency and replay expectations explicit before discussing automation.
- Do not let a green Airflow run or successful `dbt build` stand in for trust.

## Internal source notes

- [sources/ingesting-with-airflow.md](sources/ingesting-with-airflow.md)
- [sources/customizing-airflow.md](sources/customizing-airflow.md)
- [sources/snowflake-101.md](sources/snowflake-101.md)
- [sources/building-dbt.md](sources/building-dbt.md)
- [sources/batch-processing.md](sources/batch-processing.md)
- [sources/stream-processing.md](sources/stream-processing.md)
- [sources/data-quality.md](sources/data-quality.md)
- [sources/data-automation.md](sources/data-automation.md)
