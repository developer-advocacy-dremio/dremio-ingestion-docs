# Orchestration Options

As ingestion workflows grow beyond one-off jobs, orchestration becomes a
critical part of operating a Dremio Apache Polaris lakehouse reliably.

Orchestration coordinates *when* ingestion runs, *in what order*, and *under
what conditions*. It ensures that data arrives on time, failures are handled
predictably, and downstream consumers see consistent results.

This page explains why orchestration is needed and compares common orchestration
approaches used with Dremio.


## Why Orchestration Is Necessary

In simple cases, ingestion can be triggered manually or on an ad hoc schedule.
As systems mature, this approach breaks down.

Orchestration is required when:

- Ingestion depends on upstream systems or file arrival
- Multiple ingestion steps must run in sequence
- Failures need retries, alerts, or recovery
- Incremental ingestion relies on state or watermarks
- Downstream transformations depend on ingestion completion

Without orchestration, ingestion becomes brittle, opaque, and hard to scale.


## Common Orchestration Responsibilities

Regardless of tooling, orchestration typically handles:

- Scheduling (time-based or event-based)
- Dependency management between ingestion steps
- Retry and failure handling
- Environment separation (dev, staging, prod)
- Observability and logging
- Parameterization and configuration

Dremio itself executes ingestion work, but orchestration decides *when* and
*how* that work runs.


## Airflow

Apache Airflow is the most common general-purpose orchestrator used with
Dremio.

### When to Use Airflow

Airflow is a strong fit when:

- You already operate Airflow in your organization
- Ingestion spans many systems and tools
- Workflows are complex and multi-step
- You need rich dependency graphs and scheduling
- You want strong observability and retry semantics

Airflow is frequently used to orchestrate:

- CTAS and INSERT SELECT jobs
- COPY INTO batch ingestion
- DremioFrame-based ingestion
- Downstream transformations and exports

### Considerations

Airflow introduces operational overhead. It requires infrastructure,
maintenance, and operational maturity. It is best suited for teams that already
depend on it.

```python
# ============================================================
# Apache Airflow DAG for Orchestrating Dremio Ingestion
# ============================================================
#
# This DAG illustrates a common, production-style orchestration
# pattern for Dremio using Airflow.
#
# The goal of this DAG is to:
# 1. Ingest raw data into Dremio (file-based ingestion)
# 2. Materialize an Iceberg table using CTAS
# 3. Perform incremental ingestion using INSERT SELECT
#
# IMPORTANT DESIGN PRINCIPLE:
# Airflow orchestrates *when* things run.
# Dremio performs *all compute and ingestion work*.
#
# Airflow should never process data itself.
# ============================================================

from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.http.operators.http import SimpleHttpOperator

from dremioframe.client import DremioClient


# ------------------------------------------------------------
# DAG DEFAULT ARGUMENTS
# ------------------------------------------------------------
# These settings control retries, failure handling,
# and scheduling behavior for all tasks in the DAG.

default_args = {
    "owner": "data-platform",
    "depends_on_past": False,
    "email_on_failure": True,
    "email_on_retry": False,
    "retries": 2,
    "retry_delay": timedelta(minutes=10),
}


# ------------------------------------------------------------
# DAG DEFINITION
# ------------------------------------------------------------
# This DAG runs once per day and processes the previous day's data.
# The schedule and start_date should align with how upstream
# systems deliver files or data.

with DAG(
    dag_id="dremio_daily_ingestion",
    description="Daily ingestion pipeline orchestrating Dremio lakehouse ingestion",
    default_args=default_args,
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False,
    max_active_runs=1,
    tags=["dremio", "ingestion", "lakehouse"],
) as dag:


    # --------------------------------------------------------
    # Helper Function: Run SQL in Dremio
    # --------------------------------------------------------
    # This function wraps SQL execution so Airflow tasks
    # remain thin and declarative.
    #
    # All ingestion logic stays in SQL, not Python.

    def run_dremio_sql(sql: str):
        client = DremioClient()
        client.query(sql)


    # --------------------------------------------------------
    # Task 1: Batch File Ingestion Using COPY INTO
    # --------------------------------------------------------
    # This task pulls raw files from object storage and
    # appends them into a raw Iceberg table.
    #
    # COPY INTO is idempotent and safe to retry.

    copy_into_raw = PythonOperator(
        task_id="copy_into_raw_orders",
        python_callable=run_dremio_sql,
        op_kwargs={
            "sql": """
                COPY INTO raw.orders
                FROM '@s3.company_data/orders/{{ ds }}/'
                FILE_FORMAT 'parquet'
                ON_ERROR 'SKIP_FILE'
            """
        },
    )


    # --------------------------------------------------------
    # Task 2: Initial Table Creation (CTAS)
    # --------------------------------------------------------
    # This task creates a curated Iceberg table if it does
    # not already exist.
    #
    # CTAS is typically run once, but it is safe to include
    # in a DAG if guarded by IF NOT EXISTS.

    create_curated_table = PythonOperator(
        task_id="create_curated_orders_table",
        python_callable=run_dremio_sql,
        op_kwargs={
            "sql": """
                CREATE TABLE IF NOT EXISTS analytics.orders
                PARTITION BY (order_date)
                AS
                SELECT
                    order_id,
                    customer_id,
                    amount,
                    CAST(order_timestamp AS DATE) AS order_date
                FROM raw.orders
                WHERE 1 = 0
            """
        },
    )


    # --------------------------------------------------------
    # Task 3: Incremental Ingestion Using INSERT SELECT
    # --------------------------------------------------------
    # This task performs incremental ingestion from the raw
    # table into the curated Iceberg table.
    #
    # The watermark logic ensures idempotency.
    # Reruns will not duplicate data.

    incremental_insert = PythonOperator(
        task_id="incremental_insert_orders",
        python_callable=run_dremio_sql,
        op_kwargs={
            "sql": """
                INSERT INTO analytics.orders
                SELECT
                    r.order_id,
                    r.customer_id,
                    r.amount,
                    CAST(r.order_timestamp AS DATE) AS order_date
                FROM raw.orders r
                WHERE r.order_timestamp >
                    (
                        SELECT COALESCE(MAX(order_timestamp),
                                        TIMESTAMP '1970-01-01 00:00:00')
                        FROM analytics.orders
                    )
            """
        },
    )


    # --------------------------------------------------------
    # Task Dependencies
    # --------------------------------------------------------
    # This defines the execution order:
    #
    # 1. Load raw files
    # 2. Ensure curated table exists
    # 3. Append incremental data
    #
    # Airflow ensures each step completes successfully
    # before moving to the next.

    copy_into_raw >> create_curated_table >> incremental_insert


# ============================================================
# KEY TAKEAWAYS
# ============================================================
#
# - Airflow controls orchestration, not computation
# - Dremio performs all ingestion and transformation work
# - SQL remains the source of truth for ingestion logic
# - COPY INTO provides safe, retryable raw ingestion
# - INSERT SELECT enables incremental, partition-aware ingestion
#
# This pattern scales well from simple batch ingestion
# to complex, multi-step lakehouse pipelines.
# ============================================================
```

## dbt

dbt is commonly used alongside Dremio for SQL-centric workflows.

### When to Use dbt

dbt is a good choice when:

- Ingestion logic is primarily SQL-based
- You want version-controlled SQL pipelines
- Transformations follow an ELT pattern
- You already use dbt for analytics engineering

With Dremio, dbt is often used to orchestrate:

- CTAS-style table creation
- Incremental INSERT SELECT pipelines
- View-based transformations layered on ingested data

### Considerations

dbt focuses on SQL transformations, not ingestion from external systems.
It is best paired with other tools for API, file, or database ingestion.

```yaml
# ============================================================
# dbt + Dremio Example
# ============================================================
#
# This example illustrates how dbt is commonly used with Dremio
# to orchestrate SQL-driven ingestion and transformation into
# Apache Iceberg tables managed by a Dremio (Polaris-based) catalog.
#
# The pattern shown here is:
# 1. dbt connects to Dremio as the SQL execution engine
# 2. dbt models define CTAS / INSERT SELECT logic
# 3. Dremio executes all compute and Iceberg writes
#
# dbt is responsible for orchestration and versioning of SQL,
# not data movement or computation.
# ============================================================


# ------------------------------------------------------------
# profiles.yml
# ------------------------------------------------------------
# dbt profile configuration for Dremio Cloud
# (typically stored in ~/.dbt/profiles.yml)

dremio:
  target: prod
  outputs:
    prod:
      type: dremio
      method: token
      token: "{{ env_var('DREMIO_PAT') }}"
      project_id: "{{ env_var('DREMIO_PROJECT_ID') }}"
      endpoint: "https://data.dremio.cloud"
      threads: 4


# ------------------------------------------------------------
# dbt_project.yml
# ------------------------------------------------------------
# High-level dbt project configuration

name: dremio_lakehouse
version: "1.0"
profile: dremio

models:
  dremio_lakehouse:
    bronze:
      materialized: table
    silver:
      materialized: incremental
    gold:
      materialized: view


# ------------------------------------------------------------
# models/bronze/orders_raw.sql
# ------------------------------------------------------------
# Initial ingestion using CTAS semantics
# This model creates a managed Iceberg table in Dremio

{{ config(materialized='table') }}

SELECT
  order_id,
  customer_id,
  amount,
  order_timestamp
FROM postgres.sales.orders


# ------------------------------------------------------------
# models/silver/orders_incremental.sql
# ------------------------------------------------------------
# Incremental ingestion pattern using INSERT SELECT semantics
# dbt manages the incremental logic
# Dremio executes the SQL and writes Iceberg snapshots

{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

SELECT
  order_id,
  customer_id,
  amount,
  CAST(order_timestamp AS DATE) AS order_date
FROM {{ ref('orders_raw') }}

{% if is_incremental() %}
WHERE order_timestamp >
  (
    SELECT COALESCE(MAX(order_timestamp),
                    TIMESTAMP '1970-01-01 00:00:00')
    FROM {{ this }}
  )
{% endif %}


# ------------------------------------------------------------
# models/gold/orders_summary.sql
# ------------------------------------------------------------
# Analytical view built on top of ingested Iceberg tables

{{ config(materialized='view') }}

SELECT
  order_date,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount
FROM {{ ref('orders_incremental') }}
GROUP BY order_date


# ============================================================
# KEY TAKEAWAYS
# ============================================================
#
# - dbt controls SQL versioning and execution order
# - Dremio executes all SQL and manages Iceberg metadata
# - CTAS is used for initial ingestion
# - Incremental models translate to INSERT SELECT patterns
# - Views sit on top of Iceberg tables for analytics
#
# dbt + Dremio works best for SQL-first ELT pipelines
# where Iceberg tables are the durable storage layer.
# ============================================================
```

## DremioFrame Built-In Orchestration

DremioFrame includes built-in orchestration capabilities designed specifically
for Dremio-centric workflows.

### When to Use DremioFrame Orchestration

DremioFrame orchestration is a strong fit when:

- Ingestion logic lives in Python
- Pipelines are Dremio-focused
- You want fewer external dependencies
- Workflows are lightweight to moderately complex
- You want tight integration with Dremio metadata and jobs

This approach is often used for:

- API ingestion pipelines
- File system ingestion
- Database ingestion
- Coordinating SQL-driven ingestion steps
- Lightweight data quality checks

### Considerations

DremioFrame orchestration is not a replacement for enterprise schedulers.
It is best suited for teams that want simplicity and tight coupling to Dremio.

```python
# ============================================================
# Built-in Orchestration with DremioFrame
# ============================================================
#
# This example illustrates how DremioFrame’s built-in orchestration
# capabilities can be used to coordinate ingestion workflows
# without relying on an external scheduler like Airflow.
#
# The pattern shown here:
# 1. Defines a simple ingestion pipeline in Python
# 2. Executes ingestion steps in sequence
# 3. Relies on Dremio for all compute and Iceberg writes
#
# This approach is best suited for:
# - Dremio-centric pipelines
# - Lightweight to medium-complexity workflows
# - Teams that want fewer moving parts
# ============================================================

from dremioframe.client import DremioClient
from dremioframe.orchestration import Pipeline, Task

# ------------------------------------------------------------
# Initialize Dremio Client
# ------------------------------------------------------------
client = DremioClient()

# ------------------------------------------------------------
# Define Task Functions
# ------------------------------------------------------------
# Each task is intentionally small and declarative.
# Heavy logic lives in SQL or in Dremio-managed ingestion.

def ingest_raw_files():
    client.query("""
        COPY INTO raw.events
        FROM '@s3.company_data/events/{{ ds }}'
        FILE_FORMAT 'parquet'
        ON_ERROR 'SKIP_FILE'
    """)

def ensure_curated_table():
    client.query("""
        CREATE TABLE IF NOT EXISTS analytics.events
        PARTITION BY (event_date)
        AS
        SELECT
            event_id,
            event_type,
            CAST(event_timestamp AS DATE) AS event_date
        FROM raw.events
        WHERE 1 = 0
    """)

def incremental_insert():
    client.query("""
        INSERT INTO analytics.events
        SELECT
            event_id,
            event_type,
            CAST(event_timestamp AS DATE) AS event_date
        FROM raw.events
        WHERE event_timestamp >
            (
                SELECT COALESCE(MAX(event_timestamp),
                                TIMESTAMP '1970-01-01 00:00:00')
                FROM analytics.events
            )
    """)

# ------------------------------------------------------------
# Define Pipeline Tasks
# ------------------------------------------------------------
# Tasks are registered explicitly and executed in order.

raw_ingestion = Task(
    name="raw_file_ingestion",
    callable=ingest_raw_files,
    retries=2
)

table_bootstrap = Task(
    name="ensure_curated_table",
    callable=ensure_curated_table
)

incremental_load = Task(
    name="incremental_insert",
    callable=incremental_insert
)

# ------------------------------------------------------------
# Build and Run the Pipeline
# ------------------------------------------------------------
# The pipeline defines execution order and failure behavior.

pipeline = Pipeline(
    name="daily_event_ingestion",
    tasks=[
        raw_ingestion,
        table_bootstrap,
        incremental_load
    ]
)

pipeline.run()

# ============================================================
# KEY TAKEAWAYS
# ============================================================
#
# - Orchestration logic lives in Python
# - SQL remains the source of truth for ingestion
# - No external scheduler is required
# - Tasks are easy to test and reason about
# - Best suited for Dremio-first ingestion workflows
#
# This approach provides a clean middle ground between
# ad hoc scripts and full enterprise schedulers.
# ============================================================
```

## Choosing the Right Orchestration Tool

The right choice depends on scope and complexity.

A common progression looks like this:

- Simple pipelines start with DremioFrame orchestration
- SQL-heavy pipelines introduce dbt
- Cross-system, enterprise pipelines move to Airflow

These tools are complementary, not mutually exclusive.


## Best Practices

### Keep Orchestration Thin

Orchestration should coordinate execution, not perform heavy computation.
Push data processing into Dremio SQL or external engines.


### Make Ingestion Idempotent

Design ingestion steps so reruns do not duplicate data. This simplifies retries
and recovery.

### Separate Ingestion from Transformation

Ingest raw data first. Apply transformations in downstream steps. This improves
debuggability and resilience.


### Use Clear Boundaries Between Steps

Explicitly separate:

- Raw ingestion
- Incremental loading
- Merges and upserts
- Aggregations and analytics layers

This makes orchestration graphs easier to reason about.


### Observe and Log Everything

Track job status, row counts, and failures. Orchestration without visibility
quickly becomes operationally expensive.


## Summary

Orchestration is essential for operating ingestion pipelines at scale.
Dremio works well with multiple orchestration strategies, from lightweight
Python-based workflows to full enterprise schedulers.

Choose the simplest tool that meets your needs today, and evolve as complexity
grows.
