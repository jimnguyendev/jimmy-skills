# Ingesting with Airflow

Internal source notes for the public Datacamping page about ingestion with Airflow.

## Page shape

The source page is organized around:

- Airflow setup
- a short concepts pointer
- workflow overview
- official Docker-based setup
- lightweight custom setup
- one example ingestion DAG
- next-step suggestions

## Core ideas

- Airflow is introduced as the orchestrator for the learning stack.
- The page intentionally keeps deep concepts light and points learners toward the official Airflow docs for architecture details.
- The practical learning goal is to get one DAG running end to end, not to master every Airflow subsystem.

## Workflow emphasis

The page shows Airflow as a dependency-aware workflow runner where one DAG coordinates a chain of ingestion and validation tasks.

The learning bias is:

- get the environment running
- trigger one DAG
- verify the data in the target store

## Official setup path

The official path on the page uses Docker Compose from the Airflow quick start.

### Main steps

1. Download `docker-compose.yaml`.
2. Initialize Airflow metadata and required bootstrap services.
3. Start the stack.
4. Open the Airflow UI on `localhost:8080`.
5. Run the DAG from the web interface.

### Commands shown on the page

```bash
docker-compose up airflow-init
docker-compose up
docker-compose ps
docker-compose down
docker-compose down --volumes --rmi all
docker-compose down --volumes --remove-orphans
```

### Runtime expectation

The page says the official stack should bring up around 7 services from the compose file.

### Login expectation

- URL: `localhost:8080`
- username: `airflow`
- password: `airflow`

## Lightweight setup pointer

The page explicitly tells readers to scroll down for a custom lightweight setup if they want something simpler and less memory-intensive.

That path is expanded in the separate Week 2 customizing page, but the main ingestion page already frames it as:

- local-only
- lower memory
- easier to understand
- enough for a first DAG

## Practice DAG

The concrete teaching artifact on the page is `data_ingestion_postgres_dag`.

### Task chain shown on the page

- `ingest_data`
- `create_table`
- `insert_data`
- `check_duplicate`
- `create_view`

### Python shape shown on the page

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime
from etl.web_log_pipeline import (
    _ingesting_web,
    _create_table,
    _insert_data,
    _check_duplicate,
    _create_view,
)

dag = DAG(
    "data_ingestion_postgres_dag",
    description="DAG to load parquet file to postgres",
    schedule_interval="@once",
    start_date=datetime(2022, 1, 1),
    tags=["dec", "dec-loading"],
)
```

The code continues by wiring each `PythonOperator` and chaining them in order.

## Verification style

One of the best practical aspects of the page is that it verifies the DAG by checking the target database directly.

### Before running the DAG

```sql
SELECT count(1) FROM web_logs;
```

Expected result in the page:

- `web_logs = 0`

### After running the DAG

```sql
SELECT count(1) FROM web_logs;
SELECT count(1) FROM web_logs_view;
```

Expected result in the page:

- `web_logs = 9`
- `web_logs_view = 9`

## Sample output data

The page also shows example rows from `web_logs`, with fields such as:

- `created-at`
- `first-name`
- `page-name`
- `page-url`
- `timestamp`
- `user-name`

This is useful because it makes the lesson concrete:

- the DAG is not only green
- it also produces inspectable target records

## What this page is really teaching

Behind the commands, the page teaches a practical orchestration pattern:

1. ingest from source
2. create the target structure
3. load data
4. run a quality or dedup step
5. expose a serving view

That makes the page a good fit for:

- first Airflow projects
- data-ingestion demos
- teaching orchestration together with basic quality checks

## Follow-up directions on the page

The page suggests moving on to:

- custom Dockerfile setup
- managed or Kubernetes-hosted Airflow
- Astronomer as another way to run Airflow
- Mage as an alternative orchestration-plus-processing tool

## Useful takeaways for the skill pack

- Keep the first Airflow lesson operational, not theoretical.
- Always pair orchestration with a target-side verification query.
- A minimal DAG should still include a correctness step before the serving layer.
- For beginner material, `@once` plus a small target table is enough to teach the execution model.
