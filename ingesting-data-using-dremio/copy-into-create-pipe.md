# COPY INTO and CREATE PIPE

[`COPY INTO`](https://docs.dremio.com/dremio-cloud/sql/commands/copy-into-table/) and [`CREATE PIPE`](https://docs.dremio.com/dremio-cloud/sql/commands/create-pipe/) are Dremio’s file-based ingestion mechanisms. They are
designed for loading data from cloud object storage into Apache Iceberg tables
managed by the Apache Polaris–based Open Catalog.

These approaches are optimized for ingesting files that already exist in
object storage and for handling repeatable, scalable batch ingestion patterns.

## What COPY INTO Does

`COPY INTO` loads data from files in object storage into an existing Iceberg
table.

Key characteristics:

- Reads files directly from object storage (S3, ADLS, GCS)
- Appends data to an Iceberg table
- Supports common file formats such as CSV, JSON, and Parquet
- Can be run manually or scheduled externally
- Tracks file ingestion state to avoid reprocessing files

`COPY INTO` is a **pull-based** ingestion model. Each execution scans the source
location for new files and loads those that have not yet been processed.


## What CREATE PIPE Does

`CREATE PIPE` defines a **managed ingestion pipeline** that continuously applies
`COPY INTO` semantics.

Key characteristics:

- Defines a persistent ingestion definition
- Automatically detects new files
- Manages file state and progress
- Can run continuously or on a schedule
- Reduces orchestration overhead

`CREATE PIPE` is best thought of as a managed wrapper around `COPY INTO` that
simplifies operational complexity.


## Common Usage Patterns

### Batch File Ingestion

Use `COPY INTO` when:

- Files arrive in batches
- Ingestion runs on a known schedule
- You already have an external scheduler
- You want explicit control over ingestion runs

This pattern is common for daily or hourly drops of files.

```sql
-- ============================================================
-- Batch File Ingestion Using COPY INTO
-- ============================================================

-- This example demonstrates a scheduled batch ingestion pattern
-- where files arrive in object storage on a daily or hourly basis.
-- An external scheduler (Airflow, cron, GitHub Actions, etc.)
-- triggers this COPY INTO statement.

-- Target Iceberg table in the Dremio Catalog
-- dremio.sales.raw.orders

-- Source location in object storage
-- s3://company-data/orders/ingestion_date=2024-01-15/

COPY INTO dremio.sales.raw.orders
FROM '@s3.company_data/orders/ingestion_date=2024-01-15/'
FILE_FORMAT 'parquet'
ON_ERROR 'SKIP_FILE';

-- Notes:
-- - COPY INTO automatically tracks which files have already
--   been ingested and will not reprocess them on subsequent runs.
-- - The ingestion_date directory aligns with table partitioning.
-- - ON_ERROR SKIP_FILE allows ingestion to continue if a file
--   is malformed, while logging failures for review.
-- - The statement can be safely re-run for retries or backfills.
```

### Continuous File Ingestion

Use `CREATE PIPE` when:

- Files arrive continuously
- You want Dremio to manage ingestion state
- You want fewer moving parts
- You want ingestion to recover automatically from failures

This pattern works well for event exports, CDC-style file drops, and streaming
systems that land files periodically.

```sql
-- ============================================================
-- Continuous File Ingestion Using CREATE PIPE
-- ============================================================

-- This example demonstrates a managed, continuous ingestion
-- pattern where Dremio automatically detects and ingests new
-- files as they arrive in object storage.

-- Target Iceberg table in the Dremio Catalog
-- dremio.events.raw.clickstream

CREATE PIPE dremio.events.raw.clickstream_pipe
AS
COPY INTO dremio.events.raw.clickstream
FROM '@s3.company_data/clickstream/'
FILE_FORMAT 'parquet'
ON_ERROR 'SKIP_FILE';

-- Notes:
-- - The pipe continuously monitors the source location for new files.
-- - Dremio tracks file ingestion state internally to ensure files
--   are ingested exactly once.
-- - Failed files can be inspected and retried without reprocessing
--   successfully ingested data.
-- - The pipe can be paused or dropped without impacting the target table.
-- - This approach eliminates the need for external schedulers.


```


## When COPY INTO and CREATE PIPE Are a Good Choice

These approaches work best when:

- Data lands in cloud object storage
- Files are immutable
- Ingestion is append-only
- You want scalable ingestion without custom code
- File arrival cadence is predictable

They are especially effective for lakehouse architectures where object storage
is the system of record.


## When Not to Use This Approach

Avoid `COPY INTO` and `CREATE PIPE` when:

- Data arrives as row-level events
- You need low-latency ingestion
- Data requires complex transformations
- Files are frequently rewritten or updated
- You need exactly-once semantics across systems

In these cases, external streaming engines or custom ingestion pipelines are a
better fit.

## Best Practices

### Organize Files by Ingestion Boundary

Structure object storage paths by date or time window. This improves ingestion
efficiency and makes operational troubleshooting easier.

Example patterns:

- `/events/date=2024-01-01/`
- `/orders/ingestion_date=2024-01-01/`


### Align Table Partitioning with File Layout

Partition Iceberg tables on columns that match file organization. This minimizes
metadata scans and improves query performance.


### Use COPY INTO for Explicit Control

Prefer `COPY INTO` when you need:

- Dry runs
- Controlled backfills
- Manual retries
- Integration with external orchestration tools


### Use CREATE PIPE to Reduce Operational Overhead

Prefer `CREATE PIPE` when:

- You want ingestion to self-manage
- File arrival is continuous
- You want consistent ingestion behavior across environments


### Monitor File Volume and Size

Avoid ingesting very small files. Large numbers of small files increase metadata
overhead and degrade performance. Use upstream compaction where possible.


## Things to Avoid

Avoid these common mistakes:

- Using COPY INTO for row-level updates
- Treating CREATE PIPE as a streaming engine
- Mixing frequently rewritten files with append-only ingestion
- Ignoring partition and file layout design
- Running COPY INTO against large, unbounded directories

These patterns often lead to performance issues and operational complexity.


## Relationship to Other Ingestion Options

`COPY INTO` and `CREATE PIPE` complement SQL-based ingestion patterns.

Common combinations include:

- COPY INTO for raw ingestion
- CTAS or INSERT SELECT for transformations
- DremioFrame for orchestration and automation
- External engines for streaming ingestion

They are a core part of a file-centric lakehouse ingestion strategy.


## Summary

`COPY INTO` and `CREATE PIPE` provide scalable, reliable ingestion from object
storage into a Dremio Apache Polaris lakehouse. They excel at batch and
near-continuous file ingestion, but they are not a replacement for streaming or
row-level ingestion systems.

Used with good file organization, partitioning, and operational discipline,
they form one of the most effective ingestion patterns in a modern lakehouse.

