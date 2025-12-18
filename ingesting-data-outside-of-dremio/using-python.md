# Using PyIceberg and Python with Apache Iceberg

PyIceberg is a Python library that provides native access to Apache Iceberg
tables and catalogs. It allows Python applications to create, read, and manage
Iceberg tables without relying on a specific query engine.

When used with a Dremio Apache Polaris–based catalog, PyIceberg enables
engine-agnostic data ingestion and table management while still allowing Dremio
to query and govern the resulting data.

This page explains what PyIceberg is, how it fits into a Dremio-based lakehouse,
and when it is an appropriate ingestion tool.


## What PyIceberg Is

PyIceberg is the official Python implementation of the Apache Iceberg API. It
operates directly against Iceberg metadata and data files, independent of any
SQL engine.

With PyIceberg, Python code can:

- Create and drop Iceberg tables
- Append data files to tables
- Manage schemas and partition specs
- Interact with Iceberg snapshots and metadata
- Read table metadata and manifests

PyIceberg does not execute SQL. It works at the Iceberg table and file level.

## Using PyIceberg with a Dremio Catalog

Dremio exposes an Iceberg REST catalog endpoint that is compatible with
PyIceberg. This allows PyIceberg clients to interact with the same catalog used
by Dremio, ensuring a shared view of tables and metadata.

Typical setup includes:

- Configuring a REST-based Iceberg catalog
- Authenticating using OAuth tokens
- Enabling credential vending for secure access to storage
- Pointing PyIceberg at the Dremio-managed warehouse

```python
# ============================================================
# PyIceberg Catalog Initialization with Dremio (Polaris-based)
# ============================================================
#
# This example shows how to configure PyIceberg to connect to
# a Dremio-managed Apache Iceberg REST catalog.
#
# Once initialized, this catalog can be used to create, read,
# and manage Iceberg tables that are immediately visible in Dremio.
# ============================================================

from pyiceberg.catalog import load_catalog

# Load the Dremio Iceberg REST catalog
catalog = load_catalog(
    "dremio",
    {
        # Iceberg REST catalog endpoint exposed by Dremio
        "uri": "https://catalog.dremio.cloud/api/iceberg",

        # OAuth token endpoint for authentication
        "oauth2-server-uri": "https://login.dremio.cloud/oauth/token",

        # OAuth access token (typically a Dremio PAT or short-lived token)
        "token": "YOUR_TOKEN_HERE",

        # The Dremio project or warehouse name
        "warehouse": "your_project_name",

        # Enable credential vending for secure access to object storage
        "header.X-Iceberg-Access-Delegation": "vended-credentials",

        # Explicitly declare the catalog type
        "type": "rest"
    }
)

# ------------------------------------------------------------
# At this point, `catalog` can be used to:
# - List namespaces and tables
# - Create and drop Iceberg tables
# - Append data files to tables
# - Inspect snapshots and metadata
#
# All tables created through this catalog will appear
# in Dremio and be fully queryable and governable.
# ------------------------------------------------------------

```

Once configured, PyIceberg can create and modify Iceberg tables that are
immediately visible and queryable in Dremio.


## Ingestion Patterns with PyIceberg

PyIceberg is primarily used for **external ingestion**, where data is written
outside of Dremio’s execution engine.

Common patterns include:

- Writing data from Python batch jobs
- Integrating with Python-based data processing libraries
- Building custom ingestion services
- Writing Iceberg tables from AI or ML pipelines

Data is typically prepared in Python and written as Parquet or Arrow-backed
files, then committed to the Iceberg table via PyIceberg.

```python
# ============================================================
# Writing Data Files and Appending to an Iceberg Table with PyIceberg
# ============================================================
#
# This example illustrates the core ingestion pattern when using
# PyIceberg:
# 1. Load an Iceberg table from the catalog
# 2. Prepare data in Python
# 3. Write data files (Parquet)
# 4. Append the files to the Iceberg table
#
# This pattern is engine-agnostic and works with any Iceberg
# REST catalog, including Dremio.
# ============================================================

from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import LongType, StringType
from pyiceberg.io.pyarrow import write_table
import pyarrow as pa
import tempfile
import os

# ------------------------------------------------------------
# Load the Iceberg Catalog
# ------------------------------------------------------------
catalog = load_catalog(
    "dremio",
    {
        "uri": "https://catalog.dremio.cloud/api/iceberg",
        "oauth2-server-uri": "https://login.dremio.cloud/oauth/token",
        "token": "YOUR_TOKEN_HERE",
        "warehouse": "your_project_name",
        "header.X-Iceberg-Access-Delegation": "vended-credentials",
        "type": "rest"
    }
)

# ------------------------------------------------------------
# Load an Existing Iceberg Table
# ------------------------------------------------------------
table = catalog.load_table("analytics.bronze.users")

# ------------------------------------------------------------
# Prepare Data in Python
# ------------------------------------------------------------
# Data can originate from APIs, files, or Python logic.

data = pa.Table.from_pydict(
    {
        "id": [1, 2, 3],
        "name": ["Alice", "Bob", "Charlie"]
    }
)

# ------------------------------------------------------------
# Write Data Files (Parquet)
# ------------------------------------------------------------
# PyIceberg writes Parquet files compatible with Iceberg specs.

with tempfile.TemporaryDirectory() as tmpdir:
    data_file_path = os.path.join(tmpdir, "users.parquet")

    write_table(
        table=data,
        path=data_file_path,
        table_metadata=table.metadata
    )

    # --------------------------------------------------------
    # Append the Data File to the Iceberg Table
    # --------------------------------------------------------
    # This commits a new snapshot to the Iceberg table.

    with table.new_append() as append:
        append.append_file(data_file_path)
        append.commit()

# ------------------------------------------------------------
# Result
# ------------------------------------------------------------
# - A new snapshot is created
# - Data is immediately visible in Dremio
# - Iceberg metadata is updated atomically
# ============================================================

```

## Relationship to Dremio SQL Ingestion

PyIceberg complements, rather than replaces, Dremio’s SQL-based ingestion.

Key differences:

- PyIceberg writes data directly at the Iceberg layer
- Dremio SQL uses the Dremio engine to write Iceberg data
- PyIceberg is engine-agnostic
- Dremio SQL provides richer transformation and optimization features

Teams often use PyIceberg for ingestion and Dremio for querying, governance, and
analytics.


## When PyIceberg Is a Good Choice

PyIceberg is a strong fit when:

- Ingestion logic is written in Python
- Data originates outside Dremio-connected sources
- You want engine independence
- You are building custom ingestion services
- You need direct control over Iceberg metadata and commits
- You are integrating Iceberg into ML or AI workflows

It is especially common in data science and platform engineering contexts.

[iceframe - a high-level wraper around pyiceberg for easier usage](https://github.com/AlexMercedCoder/iceframe)

## When PyIceberg Is Not the Right Tool

PyIceberg may not be the best option when:

- Ingestion can be expressed cleanly in SQL
- You need complex joins or transformations during ingestion
- You want Dremio-managed optimization automatically
- You are ingesting large volumes of files from object storage
- Operational simplicity is more important than flexibility

In these cases, Dremio SQL, COPY INTO, CREATE PIPE, or DremioFrame are often
better choices.


## Security and Credential Vending

When used with Dremio’s catalog, PyIceberg can take advantage of credential
vending. This allows PyIceberg to receive temporary, scoped credentials for
object storage rather than embedding long-lived secrets.

This is a best practice for production environments.

```python
# ============================================================
# Credential Vending Configuration with PyIceberg and Dremio
# ============================================================
#
# This example illustrates how credential vending is configured
# when using PyIceberg with a Dremio Apache Polaris–based catalog.
#
# Credential vending allows PyIceberg to receive short-lived,
# scoped credentials for object storage instead of embedding
# long-lived cloud secrets in application code.
#
# This is the recommended approach for production environments.
# ============================================================

from pyiceberg.catalog import load_catalog

# ------------------------------------------------------------
# Load the Iceberg REST Catalog with Credential Vending Enabled
# ------------------------------------------------------------
catalog = load_catalog(
    "dremio",
    {
        # Dremio Iceberg REST catalog endpoint
        "uri": "https://catalog.dremio.cloud/api/iceberg",

        # OAuth token service used by Dremio
        "oauth2-server-uri": "https://login.dremio.cloud/oauth/token",

        # Access token issued by Dremio (PAT or OAuth token)
        "token": "YOUR_TOKEN_HERE",

        # Dremio project / warehouse name
        "warehouse": "your_project_name",

        # ----------------------------------------------------
        # Enable Iceberg credential vending
        # ----------------------------------------------------
        # This header instructs the catalog to vend temporary
        # credentials for object storage access.
        #
        # Dremio will issue short-lived credentials (e.g. S3,
        # ADLS, or GCS credentials) scoped to the table location.
        #
        "header.X-Iceberg-Access-Delegation": "vended-credentials",

        # Explicitly declare REST catalog usage
        "type": "rest"
    }
)

# ------------------------------------------------------------
# How Credential Vending Works
# ------------------------------------------------------------
# 1. PyIceberg authenticates to the Dremio catalog using OAuth
# 2. When data files are read or written, PyIceberg requests
#    temporary storage credentials from the catalog
# 3. Dremio vends short-lived, scoped credentials
# 4. PyIceberg uses those credentials to access object storage
#
# No cloud access keys are stored in code or configuration files.
# ------------------------------------------------------------

# ------------------------------------------------------------
# Benefits
# ------------------------------------------------------------
# - Improved security posture
# - No long-lived storage credentials in Python code
# - Centralized access control through the catalog
# - Works across S3, ADLS, and GCS
#
# This model aligns with modern zero-trust lakehouse designs.
# ============================================================

```

## Best Practices

Keep PyIceberg ingestion focused on writing data, not complex transformations.
Perform heavy transformations upstream or downstream using dedicated engines.

Batch writes whenever possible to avoid excessive small files.

Align partitioning strategy with expected query patterns to ensure Dremio can
optimize access.

Use PyIceberg for ingestion, and rely on Dremio for query acceleration,
reflections, and governance.

Monitor snapshot growth and plan for snapshot expiration as part of ongoing
maintenance.


## Relationship to Other Python-Based Options

PyIceberg sits at a lower level than DremioFrame.

- PyIceberg interacts directly with Iceberg tables
- DremioFrame orchestrates ingestion using Dremio SQL and APIs
- PyIceberg is engine-agnostic
- DremioFrame is Dremio-aware

Both can coexist in the same architecture, depending on workload needs.


## Summary

PyIceberg provides a powerful, engine-independent way to ingest and manage
Apache Iceberg tables using Python. When paired with a Dremio Apache Polaris
catalog, it enables flexible ingestion while preserving a unified metadata
layer for analytics and governance.

It is best suited for custom Python-driven ingestion workflows where direct
control over Iceberg tables is required.
