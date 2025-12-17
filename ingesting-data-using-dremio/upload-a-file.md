# Uploading a File

Uploading files directly into Dremio Cloud is the simplest way to ingest data
into a Dremio Apache Polaris–based lakehouse. This option is designed for
quick starts, ad hoc analysis, and small datasets where speed and simplicity
matter more than automation.

This page explains how file uploads work, when to use them, and when to avoid
them.


## What File Upload Does

The file upload feature allows you to upload a local file through the Dremio
Cloud UI and immediately materialize it as an Apache Iceberg table in the
Open Catalog.

Key characteristics:

- The file is uploaded from your local machine
- Dremio infers the schema
- Dremio creates and manages the Iceberg table
- The table is stored in the project’s managed storage
- The table is immediately queryable in Dremio

Supported formats commonly include CSV, JSON, Parquet, and Excel-based files,
depending on project configuration.


## Typical Workflow

1. Open the Dremio Cloud UI
2. Navigate to a space or folder
3. Upload a local file
4. Review inferred schema and format settings
5. Create the table
6. Query the data immediately using SQL

Once created, the resulting table behaves like any other Iceberg table in the
catalog. It can be queried, joined, accelerated, and governed.


## When File Upload Is a Good Choice

File upload is best suited for **early-stage and low-volume use cases**.

Use this option when:

- You are exploring data for the first time
- You are validating a dataset or schema
- You are doing a demo, workshop, or proof of concept
- The dataset is small and static
- The ingestion is a one-time or infrequent task
- You want the fastest possible path to analysis

This approach is often ideal for analysts, solution engineers, and developers
who need to move quickly without building pipelines.


## When File Upload Is Not a Good Choice

File upload is **not** designed for production ingestion pipelines.

Avoid this option when:

- Data arrives continuously or on a schedule
- Files are large or numerous
- Ingestion must be automated
- You need repeatability and auditability
- Data is coming from cloud object storage
- Data is owned or produced by upstream systems

Uploading files manually does not scale and introduces operational risk when
used beyond exploratory workflows.


## Best Practices

### Treat Uploaded Tables as Temporary

Uploaded tables should usually be considered **staging or exploratory assets**.
If a dataset proves valuable, migrate ingestion to a repeatable mechanism such
as CTAS, COPY INTO, or external ingestion tools.

### Move to Object Storage Early

If files already exist in S3, ADLS, or GCS, prefer ingestion methods that read
directly from object storage. This avoids unnecessary data movement and keeps
ingestion closer to the source.

### Validate Schema Before Production Use

Schema inference is convenient, but it should not replace explicit schema
management for production datasets. Review inferred types carefully.

### Avoid Using Upload for Frequent Updates

If you find yourself re-uploading newer versions of the same file, this is a
signal that you should switch to a pipeline-based ingestion approach.


## Relationship to Other Ingestion Options

File upload is the simplest ingestion path, but it is also the most limited.

As requirements grow, teams typically move to:

- CTAS or INSERT SELECT for SQL-driven ingestion
- COPY INTO or CREATE PIPE for file-based batch ingestion
- DremioFrame for automated ingestion workflows
- External engines for large-scale or streaming ingestion

File upload should be viewed as an **on-ramp**, not a destination.


## Summary

Uploading files into Dremio Cloud is a fast and approachable way to get data
into an Apache Polaris–backed lakehouse. It excels at exploration and demos,
but it is not intended for scalable or automated ingestion.

Use it to move quickly. Replace it once ingestion becomes operational.
