# Customizing Airflow

Internal source notes for the public Datacamping page about the no-frills Airflow setup.

## Page intent

This page exists to make Airflow easier to run on a local machine.

Its practical goal is:

- remove non-essential services
- use `LocalExecutor`
- keep Docker as the runtime
- make the stack light enough for learning and iteration

## Main differences from the official stack

The page explicitly calls out these changes:

- remove `redis`
- remove `worker`
- remove `triggerer`
- remove `flower`
- remove `airflow-init`
- switch from `CeleryExecutor` to `LocalExecutor`
- add `.env` for parameterization
- add a simple `entrypoint.sh` for bootstrap

## Folder and host setup

The page starts by creating a local Airflow workspace and ensuring file permissions are handled correctly.

### Host directories

```bash
mkdir -p ./dags ./logs ./plugins
```

### `AIRFLOW_UID`

The page explains that files written inside `dags`, `logs`, and `plugins` can otherwise be created as root.

Example shown:

```bash
echo -e "AIRFLOW_UID=$(id -u)" >> .env
```

Fallback example shown on the page:

```dotenv
AIRFLOW_UID=50000
```

## `.env` configuration

The page gives a concrete `.env` example for the lightweight stack.

```dotenv
POSTGRES_USER=airflow
POSTGRES_PASSWORD=airflow
POSTGRES_DB=airflow

AIRFLOW__CORE__EXECUTOR=LocalExecutor
AIRFLOW__SCHEDULER__SCHEDULER_HEARTBEAT_SEC=10

AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
AIRFLOW_CONN_METADATA_DB=postgres+psycopg2://airflow:airflow@postgres:5432/airflow
AIRFLOW_VAR__METADATA_DB_SCHEMA=airflow

AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION=True
AIRFLOW__CORE__LOAD_EXAMPLES=False
```

### Why these settings matter

- `LocalExecutor` keeps the stack single-node and simpler.
- turning off example DAGs keeps the UI focused.
- pausing DAGs at creation reduces accidental runs.
- explicit metadata DB wiring avoids hidden defaults.

## Dockerfile pattern

The page recommends an extended Airflow image instead of ad hoc container mutation.

### Dockerfile shape shown on the page

```dockerfile
FROM apache/airflow:2.9.1

ENV AIRFLOW_HOME=/opt/airflow

USER root
RUN apt-get update -qq && apt-get install vim -qqq unzip -qqq

COPY requirements.txt .

USER airflow
RUN pip install --no-cache-dir -r requirements.txt

SHELL ["/bin/bash", "-o", "pipefail", "-e", "-u", "-x", "-c"]

USER root
WORKDIR $AIRFLOW_HOME
COPY scripts scripts
RUN chmod +x scripts

USER $AIRFLOW_UID
```

### What this pattern teaches

- extend the base image once
- keep dependencies in `requirements.txt`
- package bootstrap scripts in the image
- make the environment reproducible

## Docker Compose pattern

The custom compose file is reduced to three services.

### Services preserved from the page

- `postgres`
- `scheduler`
- `webserver`

### Compose shape shown on the page

```yaml
version: "3"
services:
  postgres:
    image: postgres:13
    env_file:
      - .env
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data

  scheduler:
    build: .
    command: scheduler
    depends_on:
      - postgres
    env_file:
      - .env
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./scripts:/opt/airflow/scripts

  webserver:
    build: .
    entrypoint: ./scripts/entrypoint.sh
    depends_on:
      - postgres
      - scheduler
    env_file:
      - .env
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./scripts:/opt/airflow/scripts
    user: "${AIRFLOW_UID:-50000}:0"
    ports:
      - "8080:8080"
```

## Bootstrap entrypoint

The page keeps bootstrap logic in a shell script.

```bash
#!/usr/bin/env bash

airflow db upgrade
airflow db init
airflow users create -r Admin -u admin -p admin -f admin -l airflow
airflow webserver
```

### Why this matters

For local learning, this pattern reduces ceremony:

- no separate init service
- database upgrade is part of startup
- admin creation is explicit
- the UI comes up ready for use

## Runtime commands

The page shows:

```bash
docker-compose build
docker-compose -f docker-compose-nofrills.yml up -d
docker-compose ps
```

And it expects the launch output to reflect only the three fundamental containers.

## Operational tone of the page

This page is practical rather than architectural.

It is optimized for:

- local laptops
- first contact with Airflow
- fast iteration on DAG code
- reproducible setup via image plus env file

## Useful takeaways for the skill pack

- Teach a no-frills setup first when the goal is learning.
- Prefer an extended image over manual shelling into containers.
- Make bootstrap explicit with an entrypoint.
- Treat `AIRFLOW_UID` and mounted volumes as first-class setup concerns for local developer experience.
