# Ingesting Data into a Dremio Apache Polaris Lakehouse

This repository documents recommended patterns for ingesting data into a
Dremio lakehouse backed by an Apache Polaris–based Open Catalog.

The goal is to give data engineers and platform teams a clear, practical guide
to all supported ingestion paths. This includes ingestion performed directly
by Dremio, as well as ingestion performed externally using open engines and
third-party tools that write Iceberg tables into the catalog.

These docs are opinionated. They focus on real-world usage, trade-offs, and
when to choose one approach over another.


## Scope and Audience

These docs are intended for:

- Data engineers building ingestion pipelines
- Platform teams managing a Dremio lakehouse
- Architects designing Iceberg-based data platforms

This repository focuses on **ingestion**. It does not attempt to document all
Dremio features, analytics workflows, or BI usage.


## Ingesting Data Using Dremio

Dremio provides multiple native ingestion options that create and manage
Apache Iceberg tables directly in the Open Catalog. These approaches are
typically the fastest path to production for batch ingestion and analytics
workloads.

- [Uploading a File](ingesting-data-using-dremio/uploading-a-file.md)  
- [CTAS and INSERT SELECT](ingesting-data-using-dremio/ctas-insert-select.md)  
- [COPY INTO and CREATE PIPE](ingesting-data-using-dremio/copy-into-create-pipe.md)  
- [Ingesting with DremioFrame](ingesting-data-using-dremio/dremioframe.md)  
- [Orchestration Options](ingesting-data-using-dremio/orchestration-options.md)  

These approaches leverage Dremio compute, Dremio SQL, and Dremio-managed
Iceberg metadata.


## Ingesting Data Outside of Dremio

In some cases, ingestion happens outside of Dremio. This is common when
pipelines already exist, when streaming systems are involved, or when
specialized engines are required.

These approaches still write Apache Iceberg tables into the same
Apache Polaris–based catalog, allowing Dremio to query and govern the data.

- [Using Python](ingesting-data-outside-of-dremio/using-python.md)  
- [Using Apache Spark](ingesting-data-outside-of-dremio/using-apache-spark.md)  
- [Using Batch Ingestion Vendors](ingesting-data-outside-of-dremio/batch-ingestion-vendors.md)  
- [Using Streaming Ingestion Vendors](ingesting-data-outside-of-dremio/streaming-ingestion-vendors.md)  

These patterns emphasize open standards, engine interoperability, and
decoupled ingestion architectures.


## Guiding Principles

These docs follow a few core principles:

- Apache Iceberg is the table format of record  
- Apache Polaris is the system of record for catalog metadata  
- Ingestion should remain engine-agnostic where possible  
- Governance and lineage should be preserved regardless of ingestion path  

Dremio acts as both an ingestion engine and a query engine, but it is not
required for all data movement.


## How to Use This Repository

Start with the Dremio-native ingestion section if you are new to Dremio.
Move to external ingestion patterns as your architecture evolves.

Each page is designed to stand alone, with examples, best practices,
and clear guidance on when to use that approach.


## Next Steps

Each linked page will be expanded with:

- Conceptual overview
- Example workflows
- Trade-offs and limitations
- When to use this approach

Contributions and improvements are welcome.

