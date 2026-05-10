# Building dbt

Internal source notes for the public Datacamping page about building a starter dbt project.

## Page intent

This page teaches dbt as a small, working analytics-engineering project on top of Snowflake.

The practical focus is:

- understand project shape
- configure `dbt_project.yml`
- configure the profile
- run `dbt build`
- run `dbt test`
- inspect transform and analysis models

## Project structure shown on the page

The page lists a small demo project with the usual dbt layout:

```text
README.md
analyses/
data/
dbt_packages/
dbt_project.yml
logs/
macros/
models/
package.yml
seeds/
snapshots/
target/
tests/
```

### Why this matters

The project layout teaches that dbt is more than just SQL files:

- `models` for transformations
- `tests` for trust
- `seeds` for static input data
- `analyses` for exploratory or reporting SQL
- `macros` for reuse

## `dbt_project.yml`

The page uses a concise configuration:

```yaml
name: "booking_dbt"
version: "1.0.0"

profile: "booking_dbt"
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

models:
  booking_dbt:
    transform:
      schema: transform
      materialized: view
    analysis:
      schema: analysis
      materialized: view
```

## What this config is teaching

- schemas can be separated by purpose
- models can inherit defaults by folder
- dbt structure should be intentional and easy to navigate

## Profile configuration

The page uses a Snowflake profile:

```yaml
booking_dbt:
  outputs:
    dev:
      account: <account-identifier>
      database: warehouse
      password: <password>
      role: sysadmin
      schema: public
      threads: 1
      type: snowflake
      user: <user>
      warehouse: compute_dw
  target: dev
```

## Run flow shown on the page

The page gives the standard command sequence:

```bash
dbt init
dbt deps
dbt build
dbt compile
dbt test
```

## `dbt build` output shown on the page

The successful run on the page includes:

- 3 seed files
- 3 data tests
- 5 models

Named entities shown in the run output include:

- `default.bookings_1`
- `default.bookings_2`
- `default.customers`
- `bookings_1_test`
- `not_null_customer_id`
- `unique_customer_id`
- `transform.customer`
- `transform.combined_bookings`
- `transform.prepped_data`
- `analysis.hotel_count_by_day`
- `analysis.thirty_day_avg_cost`

## Why the run output matters

This page is useful because it shows dbt as an integrated flow:

- seed raw input
- test assumptions
- build transform layer
- build analysis layer

It is not framed as "write SQL and hope."

## `dbt test` output shown on the page

The page separately shows the success path for:

- `bookings_1_test`
- `not_null_customer_id`
- `unique_customer_id`

This reinforces the idea that tests are first-class, not optional decoration.

## Starter table DDL

The page also gives the base table definitions used by the exercise.

```sql
CREATE TABLE bookings_1 (
    id INTEGER,
    booking_reference INTEGER,
    hotel STRING,
    booking_date DATE,
    cost INTEGER
);

CREATE TABLE bookings_2 (
    id INTEGER,
    booking_reference INTEGER,
    hotel STRING,
    booking_date DATE,
    cost INTEGER
);

CREATE TABLE customers (
    id INTEGER,
    first_name STRING,
    last_name STRING,
    birthdate DATE,
    membership_no INTEGER
);
```

## What this page is really teaching

The page teaches dbt as a transformation and trust layer between warehouse tables and team-facing analysis.

Its operating model is:

1. prepare input tables or seeds
2. define transform models
3. run tests
4. expose analysis outputs

## Useful takeaways for the skill pack

- Keep the dbt teaching path concrete and project-based.
- Use named schemas or folders to separate transform work from analysis-facing outputs.
- Show `dbt build` and `dbt test` together so learners see the publish-gate flow.
- Treat tests as part of delivery, not as a later hardening step.
