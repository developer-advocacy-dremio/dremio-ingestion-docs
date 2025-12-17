# Ingesting Data with DremioFrame

[DremioFrame](https://github.com/developer-advocacy-dremio/dremio-cloud-dremioframe) is a Python library that provides a high-level, Pythonic interface
for interacting with Dremio Cloud and Dremio Software. While it offers broad
data engineering and administration capabilities, it is particularly useful
for ingestion workflows that need more flexibility than pure SQL, without
introducing a full external processing engine.

This page focuses specifically on DremioFrame’s ingestion-related features,
when they are a good fit, and how to use them effectively.

## What DremioFrame Is

DremioFrame acts as a client-side control layer for Dremio. It allows you to:

- Run SQL against Dremio programmatically
- Create and modify Iceberg tables
- Ingest data from external systems
- Move data into Apache Iceberg tables managed by Dremio’s Open Catalog

DremioFrame does not replace Dremio’s execution engine. Instead, it coordinates
ingestion using a mix of Dremio SQL, REST APIs, and optimized upload paths.


## Ingestion Capabilities Overview

DremioFrame supports multiple ingestion patterns, covering both ELT and ETL
workflows.

At a high level, ingestion features fall into four categories:

- API-based ingestion
- File-based ingestion
- Database ingestion
- SQL-driven ingestion into Iceberg tables

These options allow teams to ingest data regardless of where it originates,
while still landing data into a consistent Iceberg-based lakehouse.

## SQL-Driven Ingestion into Iceberg

In addition to external ingestion, DremioFrame supports SQL-driven ingestion
patterns such as:

- Creating Iceberg tables from queries
- Appending incremental data
- Performing merge and upsert operations

These patterns push execution to the Dremio engine while allowing Python-based
control and automation. This makes DremioFrame a strong fit for hybrid workflows
that mix SQL logic with application-level coordination.

```python
# ============================================================
# SQL-Driven Ingestion into Iceberg with DremioFrame
# ============================================================

# This example demonstrates how DremioFrame can be used to
# coordinate SQL-driven ingestion patterns that execute on
# the Dremio engine while being controlled from Python.

from dremioframe.client import DremioClient

# Initialize the Dremio client
client = DremioClient()

# ------------------------------------------------------------
# Example 1: CTAS (Create Table As Select)
# ------------------------------------------------------------
# Create a new Iceberg table from a query against a source.

client.query("""
    CREATE TABLE analytics.bronze.orders
    PARTITION BY (order_date)
    AS
    SELECT
        order_id,
        customer_id,
        amount,
        CAST(order_timestamp AS DATE) AS order_date
    FROM postgres.sales.orders
    WHERE order_timestamp < DATE '2024-01-01'
""")

# ------------------------------------------------------------
# Example 2: INSERT SELECT (Incremental Append)
# ------------------------------------------------------------
# Append new records into an existing Iceberg table.

client.query("""
    INSERT INTO analytics.bronze.orders
    SELECT
        order_id,
        customer_id,
        amount,
        CAST(order_timestamp AS DATE) AS order_date
    FROM postgres.sales.orders
    WHERE order_timestamp >= DATE '2024-01-01'
      AND order_timestamp <  DATE '2024-01-02'
""")

# ------------------------------------------------------------
# Example 3: COPY INTO (File-Based SQL Ingestion)
# ------------------------------------------------------------
# Load data from object storage files into an Iceberg table.

client.query("""
    COPY INTO analytics.bronze.events
    FROM '@s3.company_data/events/2024/01/15/'
    FILE_FORMAT 'parquet'
    ON_ERROR 'SKIP_FILE'
""")

# Notes:
# - All execution runs on Dremio compute
# - Python is used only for orchestration and coordination
# - SQL remains the source of truth for ingestion logic
# - This pattern works well for hybrid ELT workflows
```

## API-Based Ingestion

DremioFrame includes built-in support for ingesting data directly from REST
APIs. This is designed for cases where data originates outside traditional
databases or file systems.

Typical use cases include:

- SaaS platforms
- Public APIs
- Internal microservices
- Custom data services

API ingestion supports multiple modes such as replace, append, and merge,
allowing you to control how incoming data is applied to the target table.

This approach is especially useful when APIs are the system of record and no
intermediate storage layer exists.

```python
# ============================================================
# API-Based Ingestion with DremioFrame
# ============================================================

# This example demonstrates ingesting data directly from a REST
# API into an Apache Iceberg table managed by Dremio.
# The API is treated as the system of record.

from dremioframe.client import DremioClient

# Initialize the Dremio client
client = DremioClient()

# Ingest data from a REST API and merge it into an Iceberg table
client.ingest_api(
    url="https://api.example.com/users",
    table_name='"marketing"."bronze"."users"',
    mode="merge",          # 'replace', 'append', or 'merge'
    pk="id"                # Primary key used for merge semantics
)

# Notes:
# - 'replace' drops and recreates the table on each run
# - 'append' adds new rows without deduplication
# - 'merge' performs an upsert using the provided primary key
# - Pagination, authentication, and batching are handled internally
# - This pattern works well when APIs are the source of truth
```

## File Upload Ingestion

DremioFrame allows you to upload local files directly into Dremio as Iceberg
tables. This capability mirrors and extends the Dremio UI file upload feature.

Supported formats include CSV, JSON, Parquet, Excel, Avro, ORC, Arrow-based
formats, and others depending on installed dependencies.

File upload ingestion is best suited for:

- Development and testing
- Small to medium datasets
- One-time or infrequent ingestion
- Data exploration and prototyping

Because this approach moves data from the client machine into Dremio-managed
storage, it should not be used as a primary production ingestion mechanism for
large or frequent data loads.

```python
# ============================================================
# File Upload Ingestion with DremioFrame
# ============================================================

# This example demonstrates uploading local files from a developer
# machine into Dremio as Apache Iceberg tables. The resulting tables
# are fully managed by Dremio’s Open Catalog.

from dremioframe.client import DremioClient

# Initialize the Dremio client
client = DremioClient()

# ------------------------------------------------------------
# Example 1: Upload a CSV file
# ------------------------------------------------------------
# The file is uploaded from the local filesystem and materialized
# as a new Iceberg table in the specified space and folder.

client.upload_file(
    "data/sales.csv",
    '"sandbox"."uploads"."sales"'
)

# ------------------------------------------------------------
# Example 2: Upload an Excel file
# ------------------------------------------------------------
# Useful for analyst-driven workflows and ad hoc ingestion.

client.upload_file(
    "data/financials.xlsx",
    '"sandbox"."uploads"."financials"'
)

# ------------------------------------------------------------
# Example 3: Upload a Parquet file
# ------------------------------------------------------------
# Parquet uploads preserve schema and column types efficiently.

client.upload_file(
    "data/events.parquet",
    '"sandbox"."uploads"."events"'
)

# ------------------------------------------------------------
# Example 4: Upload with format-specific options
# ------------------------------------------------------------
# Additional options are passed through to the underlying reader.
# In this example, a specific Excel sheet is selected.

client.upload_file(
    "data/multi_sheet.xlsx",
    '"sandbox"."uploads"."sheet2_data"',
    sheet_name="Sheet2"
)

# Notes:
# - File format is inferred from the file extension when not specified
# - Uploaded data is stored in Dremio-managed storage
# - Tables created this way behave like any other Iceberg table
# - Best used for exploration, demos, and small-scale ingestion
```

## File System Ingestion

For larger file-based ingestion, DremioFrame supports ingesting multiple files
from a local directory or glob pattern.

This feature is optimized for bulk ingestion scenarios, such as:

- Migrating historical datasets
- Loading partitioned file drops
- Ingesting batches of Parquet or CSV files
- Development environments where object storage is mounted locally

Internally, this approach uses efficient Arrow-based reads and a staging upload
mechanism to handle large volumes of data with minimal memory overhead.

File system ingestion is often a stepping stone toward object storage–based
pipelines using COPY INTO or CREATE PIPE.

```python
# ============================================================
# File System Ingestion with DremioFrame
# ============================================================

# This example demonstrates ingesting multiple files from the local
# filesystem into a single Apache Iceberg table using glob patterns.
# The files may reside on local disks or on mounted network storage.

from dremioframe.client import DremioClient

# Initialize the Dremio client
client = DremioClient()

# ------------------------------------------------------------
# Example 1: Ingest a batch of Parquet files
# ------------------------------------------------------------
# All matching files are read, combined, and loaded into the
# target Iceberg table in a single ingestion operation.

client.ingest.files(
    pattern="data/sales_*.parquet",
    table_name='"warehouse"."raw"."sales"',
    write_disposition="replace"
)

# ------------------------------------------------------------
# Example 2: Append new CSV files using a recursive directory scan
# ------------------------------------------------------------
# Useful for partitioned datasets or rolling file drops.

client.ingest.files(
    pattern="logs/**/*.csv",
    table_name='"warehouse"."raw"."logs"',
    file_format="csv",
    recursive=True,
    write_disposition="append"
)

# ------------------------------------------------------------
# Example 3: Ingest historical data in batches
# ------------------------------------------------------------
# This pattern is commonly used for backfills and migrations.

client.ingest.files(
    pattern="archive/2023/**/*.parquet",
    table_name='"warehouse"."raw"."historical_events"',
    write_disposition="append"
)

# Notes:
# - Glob patterns control which files are ingested
# - Arrow-based streaming reads minimize memory usage
# - The staging upload path is used for efficient bulk loading
# - Best suited for batch ingestion and migrations
```


## Database Ingestion

DremioFrame supports ingesting data from external databases using JDBC- or
ODBC-based connectivity through underlying Python libraries.

This pattern is commonly used when:

- Data lives in operational databases
- You want to isolate analytics from production systems
- You want to materialize query results into Iceberg tables

Database ingestion typically follows an ELT-style approach, where data is read
from a source database and written into an Iceberg table managed by Dremio.

```python
# ============================================================
# Database Ingestion with DremioFrame
# ============================================================

# This example demonstrates ingesting data from an external
# operational database into an Apache Iceberg table managed
# by Dremio. The database remains the system of record.

from dremioframe.client import DremioClient

# Initialize the Dremio client
client = DremioClient()

# ------------------------------------------------------------
# Example 1: Ingest data from Postgres using a SQL query
# ------------------------------------------------------------
# The query is executed against the source database and the
# results are materialized into an Iceberg table.

client.ingest.database(
    connection_string="postgresql://user:password@db-host:5432/sales",
    query="SELECT * FROM orders WHERE updated_at >= CURRENT_DATE - INTERVAL '1 day'",
    table_name='"analytics"."bronze"."orders"'
)

# ------------------------------------------------------------
# Example 2: Replace vs append ingestion
# ------------------------------------------------------------
# Replace is typically used for dimension tables or snapshots.
# Append is used for incremental fact ingestion.

client.ingest.database(
    connection_string="postgresql://user:password@db-host:5432/sales",
    query="SELECT id, name, status FROM customers",
    table_name='"analytics"."bronze"."customers"',
    write_disposition="replace"
)

# ------------------------------------------------------------
# Example 3: Incremental database ingestion
# ------------------------------------------------------------
# Incremental logic is pushed into the source query.

client.ingest.database(
    connection_string="postgresql://user:password@db-host:5432/sales",
    query="""
        SELECT *
        FROM orders
        WHERE updated_at > (
            SELECT COALESCE(MAX(updated_at), '1970-01-01')
            FROM analytics.bronze.orders
        )
    """,
    table_name='"analytics"."bronze"."orders"'
)

# Notes:
# - Source systems are queried directly via JDBC/ODBC-compatible drivers
# - Only query results are ingested, not full tables unless specified
# - ELT-style ingestion minimizes load on operational databases
# - Best combined with partitioning and incremental filters


## dlt Integration

DremioFrame integrates with the dlt (Data Load Tool) ecosystem to support
ingestion from hundreds of prebuilt sources.

This integration enables ingestion from:

- SaaS platforms
- Cloud services
- APIs
- Common business applications

Using dlt with DremioFrame allows teams to leverage a mature ingestion ecosystem
while landing data directly into Dremio-managed Iceberg tables.

This is particularly valuable for teams that want rapid source coverage without
building and maintaining custom ingestion code.

```python
# ============================================================
# dlt Integration with DremioFrame
# ============================================================

# This example demonstrates using dlt (Data Load Tool) with
# DremioFrame to ingest data from an external source and land
# it directly into a Dremio-managed Apache Iceberg table.

import dlt
from dremioframe.client import DremioClient

# Initialize the Dremio client
client = DremioClient()

# ------------------------------------------------------------
# Define a dlt resource (example: REST API)
# ------------------------------------------------------------
# dlt handles extraction, pagination, retries, and normalization.

@dlt.resource(name="users")
def users_resource():
    import requests
    response = requests.get("https://api.example.com/users").json()
    yield from response["data"]

# ------------------------------------------------------------
# Ingest the dlt resource into Dremio
# ------------------------------------------------------------
# The target table is created or updated based on the
# write_disposition setting.

client.ingest.dlt(
    source=users_resource(),
    table_name='"marketing"."bronze"."users"',
    write_disposition="append",   # 'replace' or 'append'
    batch_size=10000
)

# Notes:
# - dlt provides connectors for many SaaS and cloud services
# - Schema evolution and batching are handled by dlt
# - Data is written directly into Iceberg tables in Dremio
# - This approach minimizes custom ingestion code
```

## When DremioFrame Is a Good Choice

DremioFrame works best when:

- Ingestion logic needs to live in Python
- Data originates outside Dremio-connected sources
- You want programmatic control without running Spark
- You need API, file, or SaaS ingestion
- You want to standardize ingestion into Iceberg tables

It is especially useful for platform teams building reusable ingestion utilities
or lightweight pipelines.


## When DremioFrame Is Not the Right Tool

DremioFrame is not ideal when:

- Ingestion must run at very high scale
- Low-latency streaming is required
- Heavy transformations are needed before ingestion
- You already operate large Spark or Flink pipelines
- Object storage ingestion can be handled directly by Dremio

In these cases, external engines or native Dremio ingestion mechanisms are often
a better fit.


## Best Practices

Treat DremioFrame as an ingestion coordinator, not a compute engine. Push heavy
processing to Dremio SQL or external systems when appropriate.

Use batch sizing when ingesting data from Python data structures to avoid memory
pressure and timeouts.

Prefer staging tables when performing complex merges or transformations. This
improves debuggability and operational safety.

Use file system ingestion and file upload for development and migration, not as
long-term production ingestion strategies.

Periodically optimize and clean up Iceberg tables after large ingestion runs to
maintain performance.


## Relationship to Other Ingestion Options

DremioFrame complements, rather than replaces, other ingestion approaches.

Common patterns include:

- DremioFrame for API and SaaS ingestion
- COPY INTO or CREATE PIPE for object storage ingestion
- CTAS and INSERT SELECT for SQL-based pipelines
- External engines for streaming and heavy processing

Used correctly, DremioFrame fills the gap between SQL-only ingestion and
full-scale data processing frameworks.

## Summary

DremioFrame provides a flexible, Python-first way to ingest data into a Dremio
Apache Polaris lakehouse. It excels at integrating external data sources,
coordinating SQL-driven ingestion, and simplifying ingestion logic without
introducing unnecessary infrastructure.

It should be viewed as a powerful ingestion companion rather than a universal
ingestion solution.
