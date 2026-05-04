# Snowflake 101

Internal source notes for the public Datacamping page about Snowflake basics.

## Page intent

The page teaches Snowflake through a simple warehouse bootstrap workflow.

The teaching sequence is:

1. connect with SnowSQL
2. create a database
3. create a warehouse
4. stage files
5. load files into a table
6. query the table
7. tear the environment down

## SnowSQL entry point

The page starts with a direct CLI example:

```bash
snowsql -a "RDZDWTP-ZP23882" -u longddl
```

The output shown on the page places the learner into an interactive SnowSQL session.

## Create a first database

The page creates a demo database called `sf_tuts`.

```sql
CREATE OR REPLACE DATABASE sf_tuts;
```

The point is not only to create an object, but also to let the learner see the basic Snowflake control flow:

- connect
- create object
- switch context
- verify output

## Create a virtual warehouse

The next core object is the virtual warehouse `sf_tuts_wh`.

```sql
CREATE OR REPLACE WAREHOUSE sf_tuts_wh WITH
    WAREHOUSE_SIZE='X-SMALL'
    AUTO_SUSPEND = 180
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED=TRUE;
```

## Why this warehouse config matters

The settings implicitly teach three important habits:

- start small
- suspend when idle
- resume automatically when needed

That makes the example cost-aware even though it is introductory.

## Stage data files

The page explains that a stage is a storage location used to load or unload data.

It distinguishes two types:

- internal stages
- external stages in cloud object storage

### Sample upload command

```text
PUT file://<file-path>/employees0*.csv @sf_tuts.public.%emp_basic;
```

### What the page teaches with this step

- file staging is a separate concept from table loading
- local file patterns can be used
- table stages are a practical beginner entry point

## Load staged files into a table

The page uses `COPY INTO` and mentions three parameters explicitly:

- `FILE_FORMAT`
- `PATTERN`
- `ON_ERROR`

### Example load command

```sql
COPY INTO emp_basic
  FROM @%emp_basic
  FILE_FORMAT = (type = csv field_optionally_enclosed_by='"')
  PATTERN = '.*employees0[1-5].csv.gz'
  ON_ERROR = 'skip_file';
```

## Query patterns shown on the page

The page uses simple SQL to prove the load worked.

```sql
SELECT * FROM emp_basic;

SELECT email
FROM emp_basic
WHERE email LIKE '%.uk';

SELECT first_name, last_name, DATEADD('day', 90, start_date)
FROM emp_basic
WHERE start_date <= '2017-01-01';
```

It also shows a direct insert example:

```sql
INSERT INTO emp_basic VALUES
  ('Clementine','Adamou','cadamou@sf_tuts.com','10510 Sachs Road','Klenak','2017-9-22'),
  ('Marlowe','De Anesy','madamouc@sf_tuts.co.uk','36768 Northfield Plaza','Fangshan','2017-1-26');
```

## Teardown

The page explicitly tears the environment down.

```sql
DROP DATABASE IF EXISTS sf_tuts;
DROP WAREHOUSE IF EXISTS sf_tuts_wh;
```

This is useful in training material because it teaches lifecycle hygiene, not only object creation.

## Quick-check artifacts

The page mentions practice SQL files for:

- a basic Snowflake SQL exercise on `ny_taxi_trips`
- a basic Snowflake ML exercise

## What this page is really teaching

Even though the examples are simple, the page encodes a practical warehouse workflow:

1. provision compute
2. create storage boundary
3. place files into a stage
4. load files using a controlled format and pattern
5. verify with SQL
6. clean up

## Useful takeaways for the skill pack

- Teach Snowflake with one small working path before discussing advanced architecture.
- Pair every load step with validation queries.
- Keep warehouses small and auto-suspending for learning environments.
- Use stage plus `COPY INTO` as the first mental model for warehouse loading.
