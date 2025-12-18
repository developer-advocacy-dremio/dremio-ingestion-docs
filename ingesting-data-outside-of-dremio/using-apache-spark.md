# Ingesting Data with Apache Spark

Apache Spark is one of the most widely used distributed data processing engines
for large-scale batch and streaming workloads. In lakehouse architectures,
Spark is commonly used to ingest, transform, and write data into Apache Iceberg
tables that are later queried and governed by platforms like Dremio.

This page explains what Spark is, how it fits into a Dremio Apache Polaris–based
lakehouse, and when it is an appropriate ingestion tool.


## What Apache Spark Is

Apache Spark is a distributed compute engine designed for parallel data
processing across clusters. It provides APIs for SQL, DataFrames, streaming,
and machine learning.

In the context of Iceberg-based lakehouses, Spark is often used to:

- Read data from files, databases, and streams
- Perform large-scale transformations
- Write data into Iceberg tables
- Manage complex ingestion pipelines

Spark operates independently of Dremio. It writes Iceberg tables directly via
the Iceberg API and catalog, while Dremio focuses on query acceleration,
governance, and analytics.

## Using Spark with a Dremio Catalog

Dremio exposes an Iceberg REST catalog that is compatible with Spark. This
allows Spark jobs to interact with the same catalog used by Dremio, ensuring a
shared and consistent view of tables and metadata.

Key characteristics of this integration:

- Spark uses Iceberg’s RESTCatalog
- Authentication is handled via OAuth and token exchange
- Credential vending provides secure access to object storage
- Tables written by Spark are immediately visible in Dremio

```python
# ============================================================
# Spark Session and Dremio Iceberg Catalog Configuration
# ============================================================
#
# This example shows how to configure Apache Spark to use
# Dremio’s Apache Polaris–based Iceberg REST catalog.
#
# With this configuration:
# - Spark writes Iceberg tables via the REST catalog
# - Authentication uses OAuth token exchange
# - Storage access uses credential vending
# - Tables are immediately visible in Dremio
#
# This configuration is typically placed at SparkSession
# initialization time.
# ============================================================

import os
import pyspark
from pyspark.sql import SparkSession

# ------------------------------------------------------------
# Required Environment Variables
# ------------------------------------------------------------
# DREMIO_PAT must be set in the environment running Spark.
# This should be a Dremio Personal Access Token.

DREMIO_CATALOG_URI = "https://catalog.dremio.cloud/api/iceberg"
DREMIO_AUTH_URI = "https://login.dremio.cloud/oauth/token"
DREMIO_PAT = os.environ.get("DREMIO_PAT")

CATALOG_NAME = "first-project"  # Dremio project name

if not DREMIO_PAT:
    raise ValueError("DREMIO_PAT environment variable must be set")

# ------------------------------------------------------------
# Spark Configuration
# ------------------------------------------------------------
# The Iceberg runtime and Dremio OAuth auth manager
# must be available on the Spark classpath.

conf = (
    pyspark.SparkConf()
        .setAppName("DremioIcebergSparkApp")

        # Iceberg runtime + cloud + Dremio OAuth auth manager
        .set(
            "spark.jars.packages",
            ",".join([
                "org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.9.2",
                "org.apache.iceberg:iceberg-aws-bundle:1.9.2",
                "com.dremio.iceberg.authmgr:authmgr-oauth2-runtime:0.0.5"
            ])
        )

        # Enable Iceberg Spark SQL extensions
        .set(
            "spark.sql.extensions",
            "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions"
        )

        # ----------------------------------------------------
        # Dremio Iceberg REST Catalog Definition
        # ----------------------------------------------------
        .set("spark.sql.catalog.dremio", "org.apache.iceberg.spark.SparkCatalog")
        .set("spark.sql.catalog.dremio.catalog-impl", "org.apache.iceberg.rest.RESTCatalog")
        .set("spark.sql.catalog.dremio.uri", DREMIO_CATALOG_URI)
        .set("spark.sql.catalog.dremio.warehouse", CATALOG_NAME)

        # Disable Spark-side catalog caching
        .set("spark.sql.catalog.dremio.cache-enabled", "false")

        # Enable credential vending
        .set(
            "spark.sql.catalog.dremio.header.X-Iceberg-Access-Delegation",
            "vended-credentials"
        )

        # ----------------------------------------------------
        # OAuth2 Authentication (Token Exchange)
        # ----------------------------------------------------
        .set(
            "spark.sql.catalog.dremio.rest.auth.type",
            "com.dremio.iceberg.authmgr.oauth2.OAuth2Manager"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.token-endpoint",
            DREMIO_AUTH_URI
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.grant-type",
            "token_exchange"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.client-id",
            "dremio"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.scope",
            "dremio.all"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.token-exchange.subject-token",
            DREMIO_PAT
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.token-exchange.subject-token-type",
            "urn:ietf:params:oauth:token-type:dremio:personal-access-token"
        )
)

# ------------------------------------------------------------
# Create Spark Session
# ------------------------------------------------------------
spark = SparkSession.builder.config(conf=conf).getOrCreate()

# ------------------------------------------------------------
# Result
# ------------------------------------------------------------
# Spark can now:
# - Create Iceberg tables using dremio.<namespace>.<table>
# - Append data to existing Iceberg tables
# - Share metadata seamlessly with Dremio
#
# Example usage (shown later in docs):
#   spark.sql("SHOW TABLES IN dremio.analytics")
# ============================================================

```


## Common Ingestion Patterns with Spark

Spark is typically used for ingestion when scale, complexity, or data volume
exceeds what SQL-only or lightweight Python tools can handle.

Common patterns include:

- Large historical backfills
- Complex multi-step transformations
- Data normalization and enrichment
- Schema alignment across sources
- Streaming ingestion that lands files or tables

Spark often serves as the ingestion and transformation engine, while Dremio
serves as the query and governance layer.

```python
# ============================================================
# Writing to Iceberg Tables from Apache Spark
# ============================================================
#
# This example illustrates common write patterns when using
# Apache Spark with a Dremio-managed Apache Iceberg catalog.
#
# Spark performs distributed computation and writes Iceberg
# data files, while the Dremio catalog manages metadata,
# snapshots, and storage credentials.
# ============================================================

from pyspark.sql import Row

# ------------------------------------------------------------
# Example 1: Create an Iceberg Table (CTAS-style)
# ------------------------------------------------------------
# This pattern is typically used for initial ingestion or
# bootstrapping a new table.

spark.sql("""
    CREATE TABLE dremio.analytics.bronze.orders
    USING iceberg
    PARTITIONED BY (order_date)
    AS
    SELECT
        order_id,
        customer_id,
        amount,
        CAST(order_timestamp AS DATE) AS order_date
    FROM parquet.`s3://company-data/raw/orders/`
""")

# ------------------------------------------------------------
# Example 2: Append Data to an Existing Iceberg Table
# ------------------------------------------------------------
# This pattern supports incremental ingestion.
# Each write creates a new Iceberg snapshot.

new_orders = [
    Row(order_id=101, customer_id=1, amount=250.00, order_date="2024-01-15"),
    Row(order_id=102, customer_id=2, amount=125.50, order_date="2024-01-15")
]

df = spark.createDataFrame(new_orders)

df.write \
  .format("iceberg") \
  .mode("append") \
  .save("dremio.analytics.bronze.orders")

# ------------------------------------------------------------
# Example 3: Insert Using Spark SQL
# ------------------------------------------------------------
# This approach mirrors INSERT SELECT semantics and is
# commonly used for incremental pipelines.

spark.sql("""
    INSERT INTO dremio.analytics.bronze.orders
    SELECT
        order_id,
        customer_id,
        amount,
        CAST(order_timestamp AS DATE) AS order_date
    FROM dremio.raw.orders
    WHERE order_timestamp >= DATE '2024-01-15'
      AND order_timestamp <  DATE '2024-01-16'
""")

# ------------------------------------------------------------
# Example 4: Overwrite a Partition
# ------------------------------------------------------------
# Useful for reprocessing a specific partition without
# rewriting the entire table.

spark.sql("""
    INSERT OVERWRITE dremio.analytics.bronze.orders
    PARTITION (order_date = DATE '2024-01-15')
    SELECT
        order_id,
        customer_id,
        amount
    FROM dremio.raw.orders
    WHERE CAST(order_timestamp AS DATE) = DATE '2024-01-15'
""")

# ------------------------------------------------------------
# Result
# ------------------------------------------------------------
# - Each write produces an Iceberg snapshot
# - Metadata is committed atomically
# - Tables are immediately visible in Dremio
# - Dremio can accelerate queries using reflections
# ============================================================

```


## Pros of Using Spark for Ingestion

Spark offers several advantages for ingestion workloads:

- High scalability for large datasets
- Mature ecosystem and tooling
- Rich APIs for batch and streaming
- Strong Iceberg support across versions
- Fine-grained control over write behavior

Spark is particularly effective for workloads that require heavy computation
before data is written into the lakehouse.


## Cons and Trade-Offs

Despite its power, Spark introduces trade-offs that should be considered.

Challenges include:

- Operational complexity
- Cluster provisioning and management
- Higher cost compared to SQL-native ingestion
- Longer startup times for jobs
- More moving parts in the architecture

For many ingestion scenarios, Spark may be more powerful than necessary.

## When Spark Is a Good Choice

Spark is a strong fit when:

- Data volumes are very large
- Transformations are complex
- Ingestion requires distributed processing
- Streaming ingestion is involved
- You already operate Spark clusters

It is commonly used by platform teams with existing Spark expertise.

## When Spark Is Not the Right Tool

Spark may not be the best option when:

- Ingestion can be expressed cleanly in SQL
- Data arrives as simple batch files
- Low operational overhead is a priority
- Pipelines are small or infrequent
- Teams want minimal infrastructure

In these cases, Dremio SQL, COPY INTO, CREATE PIPE, DremioFrame, or PyIceberg
often provide simpler solutions.


## Best Practices for Spark-Based Ingestion

### Treat Spark as a Writer, Not a Catalog

Spark should write Iceberg tables, not manage metadata policies. Catalog-level
governance should remain centralized in Dremio.


### Use Credential Vending

Always use credential vending through the catalog rather than embedding cloud
credentials in Spark jobs. This improves security and simplifies configuration.

```python
# ============================================================
# Credential Vending Configuration for Spark with Dremio Catalog
# ============================================================
#
# This example focuses specifically on how credential vending
# is enabled and used when Apache Spark writes to Iceberg tables
# through a Dremio Apache Polaris–based REST catalog.
#
# Credential vending ensures Spark never stores or manages
# long-lived cloud credentials directly.
# ============================================================

import os
import pyspark
from pyspark.sql import SparkSession

# ------------------------------------------------------------
# Required Environment Variables
# ------------------------------------------------------------
# Spark authenticates to Dremio using a Personal Access Token.
# Dremio then vends temporary object-storage credentials to Spark.

DREMIO_CATALOG_URI = "https://catalog.dremio.cloud/api/iceberg"
DREMIO_AUTH_URI = "https://login.dremio.cloud/oauth/token"
DREMIO_PAT = os.environ.get("DREMIO_PAT")
CATALOG_NAME = "first-project"

if not DREMIO_PAT:
    raise ValueError("DREMIO_PAT environment variable must be set")

# ------------------------------------------------------------
# Spark Configuration with Credential Vending Enabled
# ------------------------------------------------------------
conf = (
    pyspark.SparkConf()

        # Iceberg REST catalog configuration
        .set("spark.sql.catalog.dremio", "org.apache.iceberg.spark.SparkCatalog")
        .set("spark.sql.catalog.dremio.catalog-impl", "org.apache.iceberg.rest.RESTCatalog")
        .set("spark.sql.catalog.dremio.uri", DREMIO_CATALOG_URI)
        .set("spark.sql.catalog.dremio.warehouse", CATALOG_NAME)

        # ----------------------------------------------------
        # Enable Iceberg Credential Vending
        # ----------------------------------------------------
        # This header tells the catalog to issue temporary,
        # scoped credentials for object storage access.
        #
        # Spark never needs AWS, Azure, or GCP credentials
        # configured locally.
        .set(
            "spark.sql.catalog.dremio.header.X-Iceberg-Access-Delegation",
            "vended-credentials"
        )

        # ----------------------------------------------------
        # OAuth2 Token Exchange Configuration
        # ----------------------------------------------------
        # Spark exchanges the Dremio PAT for a short-lived
        # access token used only for catalog interaction.
        .set(
            "spark.sql.catalog.dremio.rest.auth.type",
            "com.dremio.iceberg.authmgr.oauth2.OAuth2Manager"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.token-endpoint",
            DREMIO_AUTH_URI
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.grant-type",
            "token_exchange"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.client-id",
            "dremio"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.scope",
            "dremio.all"
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.token-exchange.subject-token",
            DREMIO_PAT
        )
        .set(
            "spark.sql.catalog.dremio.rest.auth.oauth2.token-exchange.subject-token-type",
            "urn:ietf:params:oauth:token-type:dremio:personal-access-token"
        )
)

spark = SparkSession.builder.config(conf=conf).getOrCreate()

# ------------------------------------------------------------
# How Credential Vending Works (Conceptual)
# ------------------------------------------------------------
# 1. Spark authenticates to the Dremio catalog using OAuth
# 2. Spark requests table metadata or performs a write
# 3. Dremio vends short-lived storage credentials
# 4. Spark uses those credentials to read/write data files
# 5. Credentials expire automatically
#
# No cloud access keys are embedded in Spark jobs.
# ------------------------------------------------------------

# ============================================================
# Benefits
# ============================================================
#
# - Improved security posture
# - No long-lived secrets in Spark configs
# - Centralized access control via the catalog
# - Works across S3, ADLS, and GCS
#
# This model is strongly recommended for production Spark
# ingestion into Iceberg.
# ============================================================
```


### Align Partitioning with Query Patterns

Choose partition columns that align with how data is queried in Dremio. Poor
partitioning decisions can negate the benefits of Iceberg.


### Avoid Small File Explosion

Spark jobs that write frequently or in small batches can generate many small
files. Plan for compaction and use appropriate write settings.


### Separate Ingestion and Analytics

Use Spark to ingest and transform data. Use Dremio to query, accelerate, and
govern that data. Avoid overlapping responsibilities.


## Operational Considerations

Spark-based ingestion pipelines require monitoring, retry strategies, and
capacity planning. Teams should account for:

- Job failures and retries
- Backfills and reprocessing
- Schema evolution
- Snapshot growth
- Coordination with downstream consumers

These concerns reinforce the need for orchestration and clear ownership.


## Relationship to Other Ingestion Options

Spark is one ingestion option among many.

Typical comparisons:

- Spark vs Dremio SQL: flexibility vs simplicity
- Spark vs DremioFrame: scale vs ease of use
- Spark vs PyIceberg: compute engine vs lightweight writer

Many lakehouse architectures use Spark selectively rather than universally.


## Summary

Apache Spark is a powerful ingestion and transformation engine for Iceberg-based
lakehouses. When paired with a Dremio Apache Polaris catalog, it enables
large-scale, engine-agnostic ingestion while preserving a unified metadata
layer.

Spark should be used deliberately, reserved for workloads that justify its
complexity and operational cost.
