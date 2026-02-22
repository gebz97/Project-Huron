
## Chapter 1: Introduction to Apache Iceberg

### Evolution of Data Platforms

**OLTP vs. OLAP:**

| System Type | Workload | Examples | Characteristics |
|-------------|----------|----------|-----------------|
| **OLTP** | Transactional | PostgreSQL, MySQL, SQL Server | Row-based, many small reads/writes |
| **OLAP** | Analytical | Data warehouses | Columnar, complex aggregations |

### Components of an Analytical System

```
┌─────────────────────────────────────┐
│         Compute Engine              │ ← Spark, Dremio, Flink, Trino
├─────────────────────────────────────┤
│            Catalog                  │ ← Hive Metastore, AWS Glue, Nessie
├─────────────────────────────────────┤
│         Storage Engine              │ ← Manages data layout
├─────────────────────────────────────┤
│         Table Format                │ ← Iceberg, Hudi, Delta Lake
├─────────────────────────────────────┤
│         File Format                 │ ← Parquet, ORC, Avro
├─────────────────────────────────────┤
│           Storage                   │ ← S3, HDFS, ADLS, GCS
└─────────────────────────────────────┘
```

### Data Warehouse

**Pros:**
- Single source of truth
- Fast analytical queries
- Strong governance
- Schema enforcement

**Cons:**
- Vendor lock-in (proprietary formats)
- Expensive storage and compute
- Only structured data
- No native ML support
- Data copies needed for other workloads

### Data Lake

**Origins:**
- Hadoop + HDFS for cheap storage
- MapReduce for processing (complex Java jobs)
- Hive for SQL-on-Hadoop (converted SQL to MapReduce)
- Hive table format: directories + files = tables

**Hive Table Format:**
```
Table = directory + all files in it
Partitions = subdirectories
```

**Pros of Data Lakes:**
- Lower cost
- Open formats (Parquet, ORC, Avro)
- Handles unstructured data
- Supports ML use cases

**Cons of Data Lakes:**
- Poor performance (no indexes, no ACID)
- Lots of configuration needed
- No ACID transactions
- File-level changes inefficient
- No atomic multi-partition updates
- Listing files is slow (object storage throttling)

### Data Lakehouse

**Definition:** Combines data lake flexibility with data warehouse functionality

**Key Benefits:**
- **Fewer copies = less drift** - No need to copy data to warehouse
- **Faster queries** - Metadata optimizations
- **Historical snapshots** - Time travel, rollback
- **Affordable** - Lower storage costs, avoid duplication
- **Open architecture** - No vendor lock-in

### What is a Table Format?

A table format answers: **"What data is in this table?"**

It provides an abstraction over physical files so multiple engines can interact with the same data consistently.

### Hive Table Format Limitations

| Problem | Why It Matters |
|---------|----------------|
| No atomic file swaps | Can't update single file safely |
| No multi-partition transactions | Inconsistent reads during updates |
| Poor concurrency | Multiple writers cause issues |
| Slow file listing | Listing millions of files kills performance |
| Derived partition columns | Users must know to filter on derived columns |
| Stale statistics | Async jobs, often out-of-date |
| Object storage throttling | Too many files in one prefix = throttled |

### Modern Table Formats (Iceberg, Hudi, Delta Lake)

**Key Improvements:**
- Track tables as canonical list of files (not directories)
- ACID transactions
- Safe concurrent writes
- Rich metadata and statistics
- Faster query planning

### Apache Iceberg History

- **2017:** Created at Netflix by Ryan Blue and Daniel Weeks
- **2018:** Donated to Apache Software Foundation
- **Contributors:** Apple, Dremio, AWS, Tencent, LinkedIn, Stripe

**Design Goals:**
- **Consistency** - Atomic multi-partition updates
- **Performance** - Avoid file listing, use metadata
- **Ease of use** - Hidden partitioning, intuitive queries
- **Evolvability** - Safe schema/partition evolution
- **Scalability** - Petabyte scale

### Iceberg Architecture Overview

```
Catalog (maps table name → current metadata file)
    ↓
Metadata File (schema, partition spec, snapshots)
    ↓
Manifest List (list of manifests for a snapshot)
    ↓
Manifest Files (list of datafiles + stats)
    ↓
Data Files (Parquet, ORC, Avro) + Delete Files
```

### Key Features of Iceberg

| Feature | Description |
|---------|-------------|
| **ACID Transactions** | Optimistic concurrency control |
| **Partition Evolution** | Change partitioning without rewriting table |
| **Hidden Partitioning** | Partition transforms, no extra columns needed |
| **Row-level Operations** | Copy-on-write or merge-on-read |
| **Time Travel** | Query historical snapshots |
| **Version Rollback** | Revert to previous state |
| **Schema Evolution** | Add, drop, rename, reorder columns safely |

**Hidden Partitioning Example:**
```sql
-- Table partitioned by hour of order_ts (no extra column needed)
CREATE TABLE orders (
  order_id BIGINT,
  order_ts TIMESTAMP
) PARTITIONED BY (HOUR(order_ts));

-- Query benefits from partitioning automatically
SELECT * FROM orders WHERE order_ts BETWEEN '2023-01-01' AND '2023-01-02';
```

---
