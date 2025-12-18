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
