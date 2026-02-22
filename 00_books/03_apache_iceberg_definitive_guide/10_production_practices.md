
# 📖 Part III: Apache Iceberg in Practice

## Chapter 10: Production Practices

### Metadata Tables

**history - Table evolution:**
```sql
-- Spark SQL
SELECT * FROM my_catalog.table.history;

-- Dremio
SELECT * FROM TABLE(table_history('catalog.table'));

-- Trino
SELECT * FROM "table$history";
```

| Field | Type | Description |
|-------|------|-------------|
| `made_current_at` | timestamp | When snapshot became current |
| `snapshot_id` | long | Unique snapshot ID |
| `parent_id` | long | Parent snapshot ID |
| `is_current_ancestor` | boolean | Part of current lineage |

**metadata_log_entries - Metadata file history:**
```sql
SELECT * FROM my_catalog.table.metadata_log_entries;
```

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | timestamp | When metadata updated |
| `file` | string | Metadata file path |
| `latest_snapshot_id` | long | Latest snapshot ID |
| `latest_schema_id` | int | Latest schema ID |
| `latest_sequence_number` | long | Sequence number |

**snapshots - Snapshot details:**
```sql
SELECT * FROM my_catalog.table.snapshots;
```

| Field | Type | Description |
|-------|------|-------------|
| `committed_at` | timestamp | When snapshot created |
| `snapshot_id` | long | Snapshot ID |
| `parent_id` | long | Parent snapshot ID |
| `operation` | string | append, overwrite, etc. |
| `manifest_list` | string | Manifest list path |
| `summary` | map | Metrics (records added, files, etc.) |

**files - Current datafiles:**
```sql
SELECT * FROM my_catalog.table.files;
```

| Field | Type | Description |
|-------|------|-------------|
| `content` | int | 0=data, 1=position delete, 2=equality delete |
| `file_path` | string | File location |
| `file_format` | string | PARQUET, AVRO, ORC |
| `spec_id` | int | Partition spec ID |
| `partition` | struct | Partition values |
| `record_count` | long | Records in file |
| `file_size_in_bytes` | long | File size |
| `column_sizes` | map | Column ID → size |
| `value_counts` | map | Column ID → count |
| `null_value_counts` | map | Column ID → null count |
| `lower_bounds` | map | Column ID → min value |
| `upper_bounds` | map | Column ID → max value |

**manifests - Manifest files:**
```sql
SELECT * FROM my_catalog.table.manifests;
```

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Manifest path |
| `length` | long | File size |
| `partition_spec_id` | int | Partition spec ID |
| `added_snapshot_id` | long | Snapshot that added this |
| `added_data_files_count` | int | Files added |
| `existing_data_files_count` | int | Existing files |
| `deleted_data_files_count` | int | Files deleted |
| `partition_summaries` | array | Partition stats |

**partitions - Partition info:**
```sql
SELECT * FROM my_catalog.table.partitions;
```

| Field | Type | Description |
|-------|------|-------------|
| `partition` | struct | Partition values |
| `spec_id` | int | Partition spec ID |
| `record_count` | long | Records in partition |
| `file_count` | int | Files in partition |

**all_data_files - Files across all snapshots:**
```sql
SELECT * FROM my_catalog.table.all_data_files;
```

**all_manifests - Manifests across all snapshots:**
```sql
SELECT * FROM my_catalog.table.all_manifests;
```

**refs - Branches and tags:**
```sql
SELECT * FROM my_catalog.table.refs;
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Reference name |
| `type` | string | BRANCH or TAG |
| `snapshot_id` | long | Snapshot ID |
| `max_reference_age_in_ms` | long | Max age |
| `min_snapshots_to_keep` | int | Min snapshots |
| `max_snapshot_age_in_ms` | long | Max snapshot age |

**entries - File operations across snapshots:**
```sql
SELECT * FROM my_catalog.table.entries;
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | int | 0=existing, 1=added, 2=deleted |
| `snapshot_id` | long | Snapshot ID |
| `sequence_number` | long | Operation order |
| `data_file` | struct | File details |

### Metadata Table Queries

**Find files for a snapshot:**
```sql
SELECT f.*, e.snapshot_id
FROM catalog.table.entries AS e
JOIN catalog.table.files AS f
ON e.data_file.file_path = f.file_path
WHERE e.status = 1 AND e.snapshot_id = <snapshot_id>;
```

**Track file history:**
```sql
SELECT snapshot_id, sequence_number, status
FROM catalog.table.entries
WHERE data_file.file_path = '<file_path>'
ORDER BY sequence_number ASC;
```

**Partition evolution tracking:**
```sql
SELECT spec_id, COUNT(*) as partition_count
FROM catalog.table.partitions
GROUP BY spec_id;
```

**Monitor branch storage:**
```sql
SELECT r.name as branch_name, e.snapshot_id, 
       SUM(f.file_size_in_bytes) as total_size_bytes
FROM catalog.table.refs AS r
JOIN catalog.table.entries AS e ON r.snapshot_id = e.snapshot_id
JOIN catalog.table.files AS f ON e.data_file.file_path = f.file_path
WHERE r.type = 'BRANCH'
GROUP BY r.name, e.snapshot_id;
```

### Table Branching (Java API)

```java
String branch = "ingestion-validation-branch";

// Create branch
table.manageSnapshots()
  .createBranch(branch, 3)
  .setMinSnapshotsToKeep(branch, 2)
  .setMaxSnapshotAgeMs(branch, 3600000)
  .setMaxRefAgeMs(branch, 604800000)
  .commit();

// Write to branch
table.newAppend()
  .appendFile(INCOMING_FILE)
  .toBranch(branch)
  .commit();

// Read from branch
TableScan branchRead = table.newScan().useRef(branch);

// Merge to main
table.manageSnapshots()
  .fastForward("main", branch)
  .commit();
```

### Table Branching (Spark SQL)

```sql
-- Create branch
ALTER TABLE my_catalog.my_db.sales_data
CREATE BRANCH ingestion-validation-branch
RETAIN 7 DAYS 
WITH RETENTION 2 SNAPSHOTS;

-- Set active branch for writes
SET spark.wap.branch = 'ingestion-validation-branch';

-- Write to branch
INSERT INTO my_catalog.my_db.sales_data VALUES (...);
```

### Table Tagging

```java
String tag = "end-of-quarter-Q3FY23";
table.manageSnapshots()
  .createTag(tag, 8)
  .setMaxRefAgeMs(tag, 86400000) -- 1 day
  .commit();

// Read from tag
TableScan tagRead = table.newScan().useRef(tag);
```

```sql
-- Create tag
ALTER TABLE catalog.db.closed_invoices 
CREATE TAG 'end-of-quarter-Q3FY23' 
AS OF VERSION 8 
RETAIN 14 DAYS;
```

### Catalog Branching (Nessie)

```sql
-- Create branch
CREATE BRANCH IF NOT EXISTS weekly_ingest_branch IN catalog;

-- Switch to branch
USE REFERENCE weekly_ingest_branch IN catalog;

-- Ingest data
INSERT INTO table_name (...);

-- Merge to main
MERGE BRANCH weekly_ingest_branch INTO main IN catalog;
```

### Catalog Tagging (Nessie)

```sql
-- Create tag
CREATE TAG IF NOT EXISTS Q1_end_snapshot IN catalog;

-- Switch to tag
USE REFERENCE Q1_end_snapshot IN catalog;

-- Query historical data
SELECT * FROM table_name;
```

### Multitable Transactions (Nessie)

```sql
-- Create branch
CREATE BRANCH IF NOT EXISTS etl IN catalog;

-- Switch to branch
USE REFERENCE etl IN catalog;

-- Run transactions on multiple tables
INSERT INTO catalog.db.tableA ...;
INSERT INTO catalog.db.tableB ...;

-- Merge all changes atomically
MERGE BRANCH etl INTO main IN catalog;
```

### Table Rollbacks

**rollback_to_snapshot:**
```sql
CALL catalog.system.rollback_to_snapshot('orders', 12345);
```

**rollback_to_timestamp:**
```sql
CALL iceberg.system.rollback_to_timestamp(
  'db.orders', 
  timestamp('2023-06-01 00:00:00')
);
```

**set_current_snapshot:**
```sql
CALL iceberg.system.set_current_snapshot('db.inventory', 123456789);
```

**cherrypick_snapshot:**
```sql
CALL iceberg.system.cherrypick_snapshot('db.products', 987654321);
```

### Catalog Rollbacks (Nessie)

```sql
-- Find commit hash
SHOW LOG nessie.main;

-- Rollback session (temporary)
SET REF nessie.main TO 'commitHash';

-- Permanent rollback
ASSIGN nessie.main AT 'commitHash';
```

---
