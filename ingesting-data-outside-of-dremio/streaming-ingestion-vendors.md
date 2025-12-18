# Streaming Ingestion into Apache Iceberg with REST Catalogs

Streaming ingestion enables near-real-time data to be written into Apache Iceberg tables as events are produced. This approach is used when data freshness matters, when systems emit continuous change events, or when teams want to unify operational and analytical workloads.

With the Apache Iceberg REST Catalog, streaming platforms can ingest into Iceberg tables without being tied to a specific metastore or engine—unlocking full interoperability in open lakehouse architectures like Dremio.


## Why Streaming Ingestion?

Streaming ingestion is appropriate when:

- Data is produced continuously
- Latency requirements are minutes or seconds
- Data originates from event streams or CDC pipelines
- You want analytics-ready data without batch delays
- You want to decouple producers from downstream consumers

Apache Iceberg makes this feasible with snapshot isolation, schema evolution, and atomic commits.


## Core Streaming Patterns

### 1. Connector-Based Streaming

**Examples**: Kafka Connect Iceberg Sink, Pulsar IO Iceberg Sink

A connector consumes from a stream and periodically commits to Iceberg tables.

**Pros**
- Low code or no code
- Mature operational model
- Built-in fault tolerance

**Cons**
- Micro-batch latency
- Limited transformation logic

**Best Practices**
- Use append-only mode when possible
- Tune commit intervals
- Monitor small-file creation
- Centralize catalog access via REST


### 2. Broker-Native Streaming

**Examples**: Redpanda Iceberg Topics, Kafka Tiered Storage (experimental)

Streaming platforms write directly to Iceberg without connectors.

**Pros**
- Lowest ingestion latency
- Fewer components to manage

**Cons**
- Append-only in most cases
- Platform-specific implementations

**Best Practices**
- Design topics with Iceberg in mind
- Use external REST catalogs for interoperability
- Separate stream and table retention logic


### 3. Stream Processing Engines

**Examples**: Apache Flink, RisingWave, Spark Structured Streaming

Stream processors handle ingestion and transformation before writing to Iceberg.

**Pros**
- Advanced transformation capabilities
- Exactly-once semantics

**Cons**
- Operational overhead
- Higher cost

**Best Practices**
- Commit only on checkpoints
- Use Iceberg v2
- Minimize commit frequency


## Streaming Vendors: OSS vs Commercial

### 🔓 Open Source / OSS-Compatible

#### Apache SeaTunnel

- OSS data integration tool with Iceberg sink
- REST catalog support available

**Best for**: Teams that want config-first pipelines with full control



#### Apache Flink + Iceberg Sink

- Rich transformation and event-time processing
- REST catalog compatible

**Best for**: High-complexity pipelines needing upserts and joins



#### RisingWave (Streaming SQL)

- Streaming database with native Iceberg sink
- SQL-first workflows

**Best for**: Teams building curated tables from real-time data



#### StreamNative / Pulsar + Iceberg Sink

- Pulsar-native ingestion with optional managed connectors
- REST catalog support depends on version

**Best for**: Pulsar-based architectures



### 💼 Managed / Commercial Solutions

#### Confluent Kafka Connect + Iceberg Sink

- Works with self-managed and Confluent Cloud
- REST catalog compatible

**Best for**: Kafka users looking for simple ingestion with low ops



#### Confluent Tableflow (Managed)

- Managed ingestion to Iceberg
- Built-in and external REST catalog support (e.g., Polaris)

**Best for**: Kafka-first teams wanting streaming-to-table without code



#### Aiven for Kafka Connect

- Managed Kafka Connect with Iceberg support
- Works with AWS Glue, Polaris, and others

**Best for**: Kafka Connect users who want managed operations


#### Redpanda + Iceberg Topics

- Broker-native Iceberg file generation
- REST catalog integration supported

**Best for**: Teams that want minimal ingestion infrastructure


## Quick Vendor Fit Guide

| Use Case | Recommended Option |
|----------|---------------------|
| Lowest ops, Kafka-centric | Confluent Tableflow |
| Low-code ingestion | Kafka Connect (self-managed or Aiven) |
| Broker-native stream-to-Iceberg | Redpanda |
| In-stream SQL transforms | RisingWave |
| Pulsar ecosystems | StreamNative |
| OSS pipeline flexibility | Apache SeaTunnel |


## Common Streaming Challenges

### Small Files
- Tune batch and checkpoint sizes
- Run compaction jobs

### Schema Evolution
- Use schema registries
- Avoid frequent type changes

### Exactly-Once Semantics
- Use checkpoint-aware sinks
- Prefer idempotent writes

### Operational Complexity
- Use managed services
- Monitor for commit failures and lag


## When Not to Stream

Avoid streaming ingestion if:

- Data arrives in predictable batches
- Latency tolerance is in hours
- Ingestion is rare or non-critical

Use batch tools like `COPY INTO`, CTAS, or scheduled jobs instead.


## Closing Thoughts

Streaming ingestion into Apache Iceberg unlocks real-time analytics without losing openness or interoperability. Whether you prefer connectors, streaming SQL, or native brokers, the REST Catalog ensures your architecture stays flexible and future-ready—especially when combined with platforms like Dremio.

