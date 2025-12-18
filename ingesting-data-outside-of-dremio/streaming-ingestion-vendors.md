# Streaming Ingestion into Apache Iceberg

Streaming ingestion enables near-real-time data to be written into Apache
Iceberg tables as events are produced. This approach is used when data freshness
matters, when systems emit continuous change events, or when teams want to unify
operational and analytical workloads.

With the introduction of the Iceberg REST Catalog, streaming platforms can now
write to Iceberg tables in a fully decoupled and interoperable way, without being
tied to a specific metastore or query engine, thus they can ingest into Dremio Catalog.

## Why Use Streaming Ingestion

Streaming ingestion is appropriate when:

- Data is produced continuously
- Latency requirements are minutes or seconds, not hours
- Data originates from event streams or CDC pipelines
- You want analytics-ready data without batch delays
- You want to decouple producers from downstream consumers

Iceberg makes streaming ingestion practical by providing snapshot isolation,
schema evolution, and consistent reads while data is being appended.


## Core Streaming Ingestion Patterns

### Connector-Based Streaming

Examples:
- Kafka Connect Iceberg Sink
- Pulsar IO Iceberg Sink

In this pattern, a connector consumes events from a stream and periodically
commits them into Iceberg tables.

**Pros**
- Low code or no code
- Mature operational model
- Built-in retries and fault tolerance
- Easy to operate in managed services

**Cons**
- Micro-batch latency
- Less flexibility for complex transformations
- Commit frequency must be tuned carefully

**Best Practices**
- Use append-only mode when possible
- Tune commit intervals to balance latency and file sizes
- Align partitions with event time
- Monitor small-file growth
- Centralize catalog access via REST


### Native Broker-to-Iceberg Integration

Examples:
- Redpanda Iceberg Topics
- Emerging Kafka Tiered Storage integrations

In this pattern, the streaming platform itself writes Iceberg files and commits
metadata directly.

**Pros**
- Minimal moving parts
- Lowest possible ingestion latency
- No separate consumers or connectors
- Strong consistency guarantees

**Cons**
- Usually append-only
- Limited transformation capabilities
- Platform-specific implementations
- Often enterprise or paid features

**Best Practices**
- Enable schema registry integration
- Design topics with Iceberg in mind from day one
- Separate stream retention from Iceberg retention
- Use REST catalogs for multi-engine interoperability
- Avoid relying on broker-level ingestion for complex logic

### Stream Processing Engines

Examples:
- Apache Flink
- RisingWave
- Spark Structured Streaming

In this pattern, a stream processor reads from streams, applies transformations,
and writes to Iceberg tables.

**Pros**
- Powerful transformations and joins
- Support for upserts and CDC
- Exactly-once semantics
- SQL-based pipelines

**Cons**
- Higher operational complexity
- Requires managing compute clusters
- Higher cost than connector-only approaches

**Best Practices**
- Use Iceberg v2 tables
- Commit on checkpoints only
- Avoid frequent commits
- Design primary keys carefully for upserts
- Keep transformation logic close to ingestion


## REST Catalog as a Streaming Enabler

The Iceberg REST Catalog is a critical enabler for streaming ingestion.

It provides:

- Centralized metadata management
- Stateless ingestion writers
- Secure token-based authentication
- Interoperability across engines
- Cloud-native deployment

Using REST catalogs allows streaming jobs, batch jobs, and query engines to all
share the same Iceberg tables safely.

**Best Practices**
- Use a single REST catalog per environment
- Isolate projects or tenants logically
- Ensure catalog availability and low latency
- Monitor commit concurrency limits
- Use credential vending where supported

## Common Challenges

### Small Files

Streaming writes naturally produce many small files.

Mitigation strategies:
- Increase batch or checkpoint sizes
- Use periodic compaction jobs
- Align partitions with ingestion cadence


### Schema Evolution

Streaming data often evolves.

Mitigation strategies:
- Use schema registries
- Allow additive schema changes
- Avoid frequent type changes
- Validate schemas before commits


### Exactly-Once Guarantees

Not all streaming paths provide true exactly-once semantics.

Mitigation strategies:
- Use checkpoint-aware sinks
- Prefer append-only ingestion
- Use idempotent writes
- Validate commit behavior during failure testing


### Operational Complexity

Streaming pipelines are long-running systems.

Mitigation strategies:
- Use managed services where possible
- Monitor lag, commit duration, and failures
- Alert on stalled pipelines
- Automate restarts and scaling


## When Not to Use Streaming Ingestion

Streaming ingestion is not always the right choice.

Avoid it when:
- Data arrives in predictable batches
- Latency requirements are relaxed
- Ingestion is infrequent
- Simplicity is more important than freshness

In these cases, batch ingestion using COPY INTO, CTAS, or scheduled jobs is often
more cost-effective and easier to manage.

## Streaming Vendors and Feature Sets for Iceberg REST Catalog Destinations

This section summarizes production-oriented streaming options that can land data
into Apache Iceberg while using an Iceberg REST Catalog for table metadata.

Each option differs in how much it abstracts, how much you operate, and how
quickly data becomes queryable.


### Confluent Kafka Connect + Iceberg Sink Connector

**What it is**
- Kafka Connect is Confluent’s connector runtime for moving data between Kafka
  and external systems. 
- The Apache Iceberg Kafka Connect sink writes Kafka topic data into Iceberg
  tables and can use an Iceberg REST catalog. 

**Feature set**
- Writes Kafka topics to Iceberg tables (typically Parquet files plus Iceberg metadata).
- REST catalog support via the connector’s catalog configuration. 
- Works in self-managed Connect clusters and managed environments.

**Strengths**
- Low-code operational model.
- Broad Kafka ecosystem compatibility.
- Strong fit when Kafka already exists as the event backbone.

**Tradeoffs**
- Micro-batch commits can create small files if commit intervals are too short.
- Connector tuning matters at scale.

**Best fit**
- Kafka-first stacks that want simple streaming-to-lakehouse without running a
  stream processor.


### Confluent Tableflow (Confluent Cloud)

**What it is**
- A managed Confluent Cloud feature that materializes Kafka topics or Flink tables
  as Iceberg or Delta tables. 
- Includes a built-in Iceberg REST catalog and can integrate with external catalogs
  such as Apache Polaris. 

**Feature set**
- Built-in Iceberg REST Catalog for tables created by Tableflow. 
- External catalog integration with Polaris / Snowflake Open Catalog. 
- Storage flexibility depending on product tier (BYOS and managed options).

**Strengths**
- Minimal operations.
- Fast path from Kafka topics to queryable Iceberg tables.

**Tradeoffs**
- Primarily a Confluent Cloud-centric workflow.
- You follow Tableflow’s operational and governance boundaries.

**Best fit**
- Teams that want “stream-to-table” with the least possible engineering and ops.

### Aiven for Kafka Connect (Managed)

**What it is**
- Managed Kafka Connect that includes an Iceberg sink connector option.

**Feature set**
- Explicit support for AWS Glue REST catalog and Snowflake Open Catalog (Polaris),
  plus other catalog modes depending on your setup. 
- Notes and constraints are documented (example: Glue REST requires pre-created tables). 

**Strengths**
- Managed operations for Connect.
- Clear catalog support matrix.
- Good fit for teams already using Aiven.

**Tradeoffs**
- Feature availability depends on the managed connector version and the catalog mode.
- Still a connector-based approach, so file sizing and commit tuning matter.

**Best fit**
- Kafka Connect users who want managed ops and direct REST-catalog destinations.


### Redpanda (Kafka-compatible) + Iceberg Topics

**What it is**
- A Kafka-compatible streaming platform that can materialize topics directly into
  Iceberg tables.

**Feature set**
- Strong emphasis on using an external REST catalog for production deployments.
- Concrete REST catalog configuration knobs (endpoint, OAuth2, credentials). 

**Strengths**
- Fewer moving parts than “Kafka + Connect”.
- Streaming system owns the write path, which can simplify operations.

**Tradeoffs**
- Platform-specific implementation and feature boundaries.
- Typically focuses on append-style materialization, not complex transformations.

**Best fit**
- Teams that want broker-level “stream to Iceberg” with a REST catalog and minimal
  external components.


### StreamNative / Apache Pulsar + Iceberg Sink Connector

**What it is**
- Pulsar IO sink connector that ingests from Pulsar topics and writes to Iceberg tables. 
- StreamNative Cloud can provide this as a managed connector experience. 

**Feature set**
- Pulsar-native connector deployment model.
- Catalog support depends on the connector version and underlying Iceberg libraries.

**Strengths**
- Natural fit for Pulsar-first architectures.
- Connector-based model remains low code.

**Tradeoffs**
- Smaller ecosystem than Kafka for many teams.
- Confirm catalog auth and REST support details for your specific catalog and connector version.

**Best fit**
- Pulsar users who want a low-code path to Iceberg tables without adding Kafka Connect.

### RisingWave (Streaming SQL Database)

**What it is**
- A streaming SQL system that supports streaming writes to Iceberg tables. 
- Explicit support for Iceberg REST catalogs via `catalog.type = 'rest'`.

**Feature set**
- Built-in Iceberg source and sink connectors.
- REST catalog configuration is first-class. 
- Can apply streaming SQL transformations before writing.

**Strengths**
- SQL-first development experience.
- Strong option when you need transformations, joins, and aggregations in-stream.

**Tradeoffs**
- You operate a new system (or adopt a managed offering).
- Storage and table-format constraints can apply depending on release.

**Best fit**
- Teams that want a streaming SQL layer that lands curated, query-ready tables into Iceberg.

### Apache SeaTunnel (OSS, Config-driven Pipelines)

**What it is**
- An open source data integration engine that can run on several execution engines.
- Provides an Iceberg sink connector with features like CDC mode, auto-create, and schema evolution. 

**Feature set**
- Iceberg sink connector with multi-table writes and schema evolution. 
- Public guidance and examples for REST-catalog-compatible destinations such as
  S3 Tables REST catalog flows.

**Strengths**
- OSS with a declarative configuration model.
- Good fit when you want a standardized pipeline pattern across many connectors.

**Tradeoffs**
- You operate the runtime.
- Connector capabilities vary by version, and REST-catalog auth details may require validation.

**Best fit**
- Teams that prefer OSS and config-driven pipelines, and can run a managed runtime themselves.


## Quick Selection Guidance

- **Lowest ops, Kafka-centric:** Confluent Tableflow. 
- **Low-code, broadly compatible:** Kafka Connect Iceberg sink (self-managed or managed via Aiven). 
- **Broker-native stream-to-Iceberg:** Redpanda Iceberg Topics with REST catalog.
- **Need in-stream SQL transforms:** RisingWave. 
- **Pulsar ecosystem:** StreamNative / Pulsar Iceberg sink. 
- **OSS, config-first pipelines:** Apache SeaTunnel. 

## Summary

Streaming ingestion into Apache Iceberg enables real-time analytics while
preserving the openness and reliability of the lakehouse.

Connector-based pipelines offer simplicity.
Native integrations offer minimal latency.
Stream processors offer maximum flexibility.

The right approach depends on latency needs, transformation complexity, and
operational maturity.

Using the Iceberg REST Catalog as the destination ensures that all approaches
remain interoperable, future-proof, and compatible with a multi-engine
lakehouse architecture.
