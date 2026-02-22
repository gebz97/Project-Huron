
## Chapter 1: Introduction to the Delta Lake Lakehouse Format

### Evolution of Data Systems

**Data Warehouses:**
- Purpose-built for aggregating structured data
- ACID transactions for data integrity
- Management features (backup, recovery, gated controls)
- Performance optimizations (indexes, partitioning)
- **Limitations:** Hard to scale for big data volume, variety, velocity

**Data Lakes:**
- Scalable storage repositories (HDFS, S3, ADLS, GCS)
- Handle volume, velocity, and variety of data
- File-based systems on commodity hardware
- Store data in native format (structured, semi-structured, unstructured)
- **Limitations:** BASE model (basically available, soft-state, eventually consistent)
  - No ACID guarantees
  - Processing failures leave inconsistent state
  - Can become "data swamps"

**Lakehouses (Data Lakehouses):**
- Combine best of data lakes + data warehouses
- Scalability and flexibility of data lakes
- Management features and performance of data warehouses
- Single coherent platform for all data analysis

| Feature | Data Warehouse | Data Lake | Lakehouse |
|---------|----------------|-----------|-----------|
| Data Structure | Structured only | All formats | All formats |
| ACID Transactions | Yes | No | Yes |
| Scalability | Vertical | Horizontal | Horizontal |
| Cost | High | Low | Low |
| Schema Enforcement | Yes | No | Yes |
| Time Travel | Limited | No | Yes |

### Project Tahoe to Delta Lake

- **Project Tahoe** (2017): Michael Armbrust's idea for transactional reliability on data lakes
- **Delta Lake** (2018): Renamed by Jules Damji
  - Metaphor: rivers flow into deltas, depositing sediments that create fertile ground
  - Represents convergence of data streams into managed data lake

**Early Use Cases:**
- **Comcast:** Reduced compute from 640 VMs → 64 VMs (10x reduction)
  - Jobs reduced from 84 → 3 (28x fewer)
- **Apple Information Security:** 300B+ events/day, hundreds of TB/day

### What is Delta Lake?

Open source storage layer that provides:
- ACID transactions
- Scalable metadata handling
- Unification of streaming and batch data processing

```
┌─────────────────────────────────────┐
│    BI Tools   │   Data Science   │  ML  │
├─────────────────────────────────────┤
│      Apache Spark • Flink • Trino    │
│      Presto • Hive • Druid • Athena  │
├─────────────────────────────────────┤
│           Delta Lake                 │ ← Storage Layer
├─────────────────────────────────────┤
│   HDFS  │  S3  │  ADLS  │  GCS  │ MinIO  │
└─────────────────────────────────────┘
```

### Key Features

| Feature | Description |
|---------|-------------|
| **ACID Transactions** | Atomic, consistent, isolated, durable modifications |
| **Scalable Metadata** | Handles petabyte-scale tables efficiently |
| **Time Travel** | Query previous versions by version or timestamp |
| **Unified Batch/Streaming** | Same APIs, ACID guarantees for streaming |
| **Schema Evolution/Enforcement** | Enforce schema on write, modify without breaking queries |
| **Audit History** | Detailed logs of all changes (who, what, when) |
| **DML Operations** | INSERT, UPDATE, DELETE, MERGE |
| **Open Source** | Linux Foundation, Apache 2.0 |
| **Performance** | Optimized for both ingestion and querying |
| **Ease of Use** | Same API as Parquet (just change format) |

**Ease of Use Example:**
```python
# Parquet
data.write.format("parquet").save("/tmp/parquet-table")

# Delta Lake (same API)
data.write.format("delta").save("/tmp/delta-table")
```

### Anatomy of a Delta Lake Table

```
Delta Table Directory
├── _delta_log/               ← Transaction log
│   ├── 00000000000000000000.json
│   ├── 00000000000000000001.json
│   └── ...
├── part-00000-xxx.parquet    ← Data files
├── part-00001-xxx.parquet
└── ...
```

**Components:**

| Component | Description |
|-----------|-------------|
| **Data Files** | Parquet format files containing actual data |
| **Transaction Log** | Ordered record of every transaction (JSON files in _delta_log) |
| **Metadata** | Schema, partitioning, configuration settings |
| **Schema** | Structure (columns, data types) enforced on write |
| **Checkpoints** | Periodic snapshots of transaction log (every 10 transactions) |

### Delta Transaction Log Protocol

**Goals:**
- **Serializable ACID writes** - Multiple concurrent writers with ACID semantics
- **Snapshot isolation for reads** - Consistent snapshot even with concurrent writes
- **Scalability** - Billions of partitions/files
- **Self-describing** - All metadata stored with data
- **Incremental processing** - Tail log for streaming

**File Level View:**

```
Version 0 (CREATE):
000...00.json: "add" → 1.parquet, 2.parquet

Version 1 (DELETE):
000...01.json: 
  "remove" → 1.parquet, 2.parquet (soft delete)
  "add" → 3.parquet (new file with remaining rows)
```

**Key Observations:**
- Deletes create new files (don't modify existing ones)
- Remove is "soft delete" (tombstone)
- Physical removal happens with `VACUUM`
- Time travel reads older snapshots via transaction log

### MVCC and Failure Handling

**Without Delta Lake (Partial Files Problem):**
```
t0: Table = file1, file2
t1: Job fails → partial file3 created (corrupt)
t2: Job succeeds → file3 (good), file4 created
Result: Queries see duplicates (good + corrupt file3)
```

**With Delta Lake:**
```
t0: Table = file1, file2 (log records these)
t1: Job fails → file3 created but NOT in log
     Queries still see only file1, file2
t2: Job succeeds → file3, file4 created AND in log
     Queries see only file3, file4
```

### Table Features (Protocol v2+)

**Old Model (Protocol versions):**
- Version 4 supports both generated columns AND CDF
- Connectors must implement ALL features to support ANY feature

**New Model (Table Features - Delta 2.3+):**
```sql
SHOW TBLPROPERTIES default.my_table;
-- delta.minReaderVersion = 3
-- delta.minWriterVersion = 7
-- delta.feature.deletionVectors = supported
-- delta.enableDeletionVectors = true
```

### Delta Kernel

**Problem:** Every new feature required rewriting connectors
**Solution:** Kernel abstracts protocol details, connectors build against stable APIs

**Advantages:**
- **Modularity** - Metadata logic in Kernel, connectors focus on data processing
- **Extensibility** - Upgrade Kernel version = get new features automatically

**Kernel Responsibilities:**
- Read JSON files
- Read Parquet log files
- Replay log with data skipping
- Read Parquet data and deletion vector files
- Transform data

### Delta UniForm

**Problem:** Multiple lakehouse formats (Delta, Iceberg, Hudi)
**Solution:** Generate all formats' metadata concurrently with Delta format

**Support:**
- Iceberg: Delta 3.0.0 (Oct 2023)
- Hudi: Delta 3.2.0 (May 2024)

---
