# CTAS and INSERT SELECT

[]`CREATE TABLE AS SELECT` (CTAS)](https://docs.dremio.com/dremio-cloud/sql/commands/create-table-as/) and []`INSERT INTO … SELECT`](https://docs.dremio.com/dremio-cloud/sql/commands/insert/) are the most common
SQL-native ingestion patterns in Dremio Cloud. They allow you to ingest data
into Apache Iceberg tables using standard SQL while keeping all metadata
managed in the Apache Polaris–based Open Catalog.

These patterns are widely used for batch ingestion, transformation pipelines,
and incremental data movement.

## What CTAS Does

CTAS creates a **new Iceberg table** and populates it with the results of a
`SELECT` query.

Key characteristics:

- Creates a new Iceberg table in the catalog
- Executes the query once at creation time
- Writes data using Dremio-managed compute
- Registers metadata automatically in the Open Catalog
- Supports reading from any Dremio source

CTAS is most often used for **initial ingestion** or **table bootstrapping**.

## What INSERT SELECT Does

`INSERT INTO … SELECT` appends data to an **existing Iceberg table**.

Key characteristics:

- Writes additional data files to an existing table
- Preserves existing schema and metadata
- Participates in Iceberg snapshot history
- Can be run repeatedly as part of a pipeline

This pattern is the foundation for **incremental ingestion** in Dremio.

## Common Ingestion Patterns

### Initial Load with CTAS

A typical pattern is to use CTAS for the first load:

- Read from a source system
- Apply light transformations
- Materialize a new Iceberg table

Once the table exists, all subsequent ingestion uses `INSERT SELECT`.

### Incremental Ingestion with INSERT SELECT

Incremental ingestion is usually achieved by filtering the source data.

Common approaches include:

- Timestamp-based filters  
- Monotonically increasing IDs  
- Change flags or status columns  

Example pattern:

- Track the maximum processed timestamp
- Filter the source on values greater than that watermark
- Append only new records

This keeps ingestion fast and avoids rewriting historical data.

```sql
-- ============================================
-- Incremental Ingestion Using INSERT SELECT
-- Timestamp-Based Watermark Pattern
-- ============================================

-- Target table (Iceberg table in Dremio Catalog)
-- dremio.sales.analytics.orders

-- Source table (could be a database, lake, or view)
-- postgres.sales.orders

-- --------------------------------------------
-- Step 1: Identify the current watermark
-- --------------------------------------------
-- Get the latest processed timestamp from the target table.
-- This represents the high-water mark for ingestion.
-- If the table is empty, use a default minimum timestamp.

WITH last_ingested AS (
  SELECT
    COALESCE(MAX(updated_at), TIMESTAMP '1970-01-01 00:00:00') AS last_ts
  FROM dremio.sales.analytics.orders
)

-- --------------------------------------------
-- Step 2: Insert only new or updated records
-- --------------------------------------------
INSERT INTO dremio.sales.analytics.orders
SELECT
  o.order_id,
  o.customer_id,
  o.order_status,
  o.total_amount,
  o.updated_at
FROM postgres.sales.orders o
CROSS JOIN last_ingested li
WHERE o.updated_at > li.last_ts;
```


### Partition-Aligned Inserts

Incremental inserts work best when the table is partitioned on a column that
naturally aligns with ingestion boundaries, such as:

- Date
- Event time
- Ingestion timestamp

This allows Iceberg to limit metadata scans and improves downstream query
performance.

```sql
-- ============================================================
-- Partition-Aligned Ingestion Using CTAS and INSERT SELECT
-- ============================================================

-- This example demonstrates:
-- 1. Creating a partitioned Iceberg table using CTAS
-- 2. Performing incremental inserts that align with the partition column
--    to improve write efficiency and query performance

-- ------------------------------------------------------------
-- Step 1: Initial Load with CTAS (Partitioned Table)
-- ------------------------------------------------------------
-- The table is partitioned by event_date, which aligns naturally
-- with daily ingestion boundaries.

CREATE TABLE dremio.sales.analytics.orders
PARTITION BY (event_date)
AS
SELECT
  order_id,
  customer_id,
  order_status,
  total_amount,
  CAST(event_timestamp AS DATE) AS event_date,
  event_timestamp
FROM postgres.sales.orders
WHERE event_timestamp < DATE '2024-01-01';

-- ------------------------------------------------------------
-- Step 2: Incremental Ingestion with INSERT SELECT
-- ------------------------------------------------------------
-- Each incremental run filters on the same partition column.
-- Only new partitions (or new data within recent partitions)
-- are written, minimizing metadata scans and file rewrites.

INSERT INTO dremio.sales.analytics.orders
SELECT
  o.order_id,
  o.customer_id,
  o.order_status,
  o.total_amount,
  CAST(o.event_timestamp AS DATE) AS event_date,
  o.event_timestamp
FROM postgres.sales.orders o
WHERE o.event_timestamp >= DATE '2024-01-01'
  AND o.event_timestamp <  DATE '2024-01-02';
```

## When This Approach Works Well

CTAS and INSERT SELECT are a strong choice when:

- Data arrives in batches
- Source systems are queryable
- Transformations are SQL-friendly
- Ingestion can run on a schedule
- You want minimal tooling complexity

These patterns fit naturally into ELT-style workflows and lakehouse
architectures.


## Best Practices

### Separate Initial Load from Incremental Loads

Use CTAS only once to create the table. Use INSERT SELECT for all subsequent
writes. Mixing these responsibilities leads to confusion and operational risk.


### Keep INSERT SELECT Idempotent

Design incremental queries so reruns do not duplicate data. Common techniques
include:

- Filtering strictly on greater-than conditions
- Writing to staging views before inserting
- Using Write-Audit-Publish patterns when needed

```sql
-- ============================================================
-- Idempotent Incremental Ingestion Using a Staging View
-- ============================================================

-- This example demonstrates:
-- 1. A staging view that isolates incremental source data
-- 2. An INSERT SELECT that can be safely re-run without
--    duplicating records in the target Iceberg table

-- ------------------------------------------------------------
-- Step 1: Create or Replace a Staging View
-- ------------------------------------------------------------
-- The staging view applies the incremental filter logic and
-- acts as a stable ingestion boundary.

CREATE OR REPLACE VIEW dremio.sales.staging.orders_incremental AS
SELECT
  o.order_id,
  o.customer_id,
  o.order_status,
  o.total_amount,
  o.updated_at
FROM postgres.sales.orders o
WHERE o.updated_at >
  (
    SELECT
      COALESCE(MAX(updated_at), TIMESTAMP '1970-01-01 00:00:00')
    FROM dremio.sales.analytics.orders
  );

-- ------------------------------------------------------------
-- Step 2: Idempotent INSERT SELECT from the Staging View
-- ------------------------------------------------------------
-- Because the staging view only exposes new records, this
-- INSERT can be re-run safely without duplicating data.

INSERT INTO dremio.sales.analytics.orders
SELECT
  order_id,
  customer_id,
  order_status,
  total_amount,
  updated_at
FROM dremio.sales.staging.orders_incremental;
```

### Avoid Overusing CTAS for Refreshes

Recreating tables with CTAS on every run destroys snapshot history, breaks
downstream dependencies, and increases compute costs. Prefer appends.


### Manage Small Files Proactively

Frequent INSERT operations can create many small files. Use compaction
strategies or scheduled maintenance to keep file sizes healthy.


### Use Views to Encapsulate Logic

Place complex logic in views and keep CTAS or INSERT SELECT statements simple.
This improves readability and makes ingestion logic easier to evolve.


## What Not to Do

Avoid these common mistakes:

- Using CTAS repeatedly to “refresh” data
- Appending without filters in incremental pipelines
- Embedding complex business logic directly in INSERT statements
- Treating INSERT SELECT as a streaming solution
- Ignoring partition strategy

These patterns may work initially but tend to fail at scale.


## Relationship to Other Ingestion Options

CTAS and INSERT SELECT are foundational, but they are not always sufficient.

Teams often move to:

- COPY INTO for file-based ingestion
- CREATE PIPE for continuous file ingestion
- DremioFrame for automation and orchestration
- External engines for streaming or heavy transformations

CTAS and INSERT SELECT remain the baseline against which other ingestion
approaches are compared.

## Summary

CTAS and INSERT SELECT provide a clean, SQL-first way to ingest data into a
Dremio Apache Polaris lakehouse. Used correctly, they support scalable,
incremental ingestion with minimal complexity.

They work best when paired with good partitioning, clear incrementality
boundaries, and disciplined operational practices.