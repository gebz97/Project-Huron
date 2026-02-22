
## Chapter 4: Optimizing Performance

### The "Small Files Problem"

```
❌ Many small files:        ✅ Fewer larger files:
┌──┐┌──┐┌──┐┌──┐┌──┐      ┌──────────┐
│10││12││ 8││15││11│ KB    │   56 KB  │
└──┘└──┘└──┘└──┘└──┘      └──────────┘
   5 file operations           1 file operation
   5 metadata reads            1 metadata read
```

**Costs:**
- Fixed: Reading actual data needed (unavoidable)
- Variable: File operations, metadata reads (reducible)

### Compaction

**Definition:** Rewriting many small files into fewer larger files

**Spark Actions API:**
```scala
import org.apache.iceberg.actions._

Table table = catalog.loadTable("myTable");
SparkActions.get()
  .rewriteDataFiles(table)
  .option("rewrite-job-order", "files-desc")
  .execute();
```

**Key Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `target-file-size-bytes` | Desired output file size | 512 MB |
| `max-concurrent-file-group-rewrites` | Max parallel groups |  |
| `max-file-group-size-bytes` | Max size per group |  |
| `partial-progress-enabled` | Commit as groups complete | false |
| `partial-progress-max-commits` | Max commits per job |  |
| `rewrite-job-order` | Order to process groups |  |

**Spark SQL Compaction:**
```sql
CALL catalog.system.rewrite_data_files(
  table => 'musicians',
  strategy => 'binpack',
  where => 'genre = "rock"',
  options => map(
    'target-file-size-bytes', '1073741824', -- 1GB
    'max-file-group-size-bytes', '10737418240' -- 10GB
  )
);
```

**Dremio Compaction:**
```sql
-- Basic compaction
OPTIMIZE TABLE catalog.MyTable;

-- By partition
OPTIMIZE TABLE catalog.MyTable 
FOR PARTITIONS year IN (2022, 2023);

-- With size parameters
OPTIMIZE TABLE catalog.MyTable 
REWRITE DATA (MIN_FILE_SIZE_MB=100, MAX_FILE_SIZE_MB=1000, TARGET_FILE_SIZE_MB=512);

-- Rewrite only manifests
OPTIMIZE TABLE catalog.MyTable REWRITE MANIFESTS;
```

### Compaction Strategies

| Strategy | What It Does | Pros | Cons |
|----------|--------------|------|------|
| **Binpack** | Combine files only, local sort | Fastest | No clustering |
| **Sort** | Sort by 1+ fields globally | Better clustering | Slower |
| **Z-order** | Sort by multiple fields equally | Best for multi-field filters | Slowest |

### Sorting

**Why Sort?**
- Concentrates similar values in fewer files
- Reduces files to scan for filtered queries

**Setting Sort Order:**
```sql
-- At table creation
CREATE TABLE catalog.nfl_players (
  id BIGINT,
  player_name VARCHAR,
  team VARCHAR
)
ORDER BY team;

-- After creation
ALTER TABLE catalog.nfl_teams WRITE ORDERED BY team;

-- In CTAS
CREATE TABLE catalog.nfl_teams AS
SELECT * FROM non_iceberg_teams_table ORDER BY team;

-- In INSERT
INSERT INTO catalog.nfl_teams
SELECT * FROM staging_table ORDER BY team;
```

**Sort Compaction:**
```sql
CALL catalog.system.rewrite_data_files(
  table => 'nfl_teams',
  strategy => 'sort',
  sort_order => 'team ASC NULLS LAST, name ASC NULLS FIRST'
);
```

### Z-order

**Concept:** Sort by multiple fields equally, creating quadrants

```
X axis: Age
Y axis: Height

Quadrants:
A: Age 1-50, Height 1-5
B: Age 51-100, Height 1-5
C: Age 1-50, Height 5-10
D: Age 51-100, Height 5-10

Query for Age=60, Height=6 → scans only quadrants with matching ranges
```

**Z-order Compaction:**
```sql
CALL catalog.system.rewrite_data_files(
  table => 'medical_cohort',
  strategy => 'zorder',
  sort_order => 'age,height'
);
```

### Partitioning

**Traditional Partitioning Problems:**
- Need extra derived columns
- Users must know to filter on derived columns
- Changing partition scheme requires rewrite

**Hidden Partitioning (Iceberg):**

```sql
-- Partition by month of order_ts (no extra column)
CREATE TABLE orders (
  order_id BIGINT,
  order_ts TIMESTAMP
) PARTITIONED BY (months(order_ts));

-- Query benefits automatically
SELECT * FROM orders 
WHERE order_ts BETWEEN '2023-01-01' AND '2023-01-31';
```

**Partition Transforms:**

| Transform | Description | Example |
|-----------|-------------|---------|
| `year(ts)` | Year only | `year(order_ts)` |
| `months(ts)` | Year + month | `months(order_ts)` |
| `days(ts)` | Year + month + day | `days(order_ts)` |
| `hours(ts)` | Year + month + day + hour | `hours(order_ts)` |
| `bucket(N, col)` | Hash to N buckets | `bucket(24, zip)` |
| `truncate(L, col)` | Truncate to L chars | `truncate(name, 1)` |

**Bucket Transform Example:**
```sql
-- Partition by zip code into 24 buckets
CREATE TABLE voters (
  voter_id BIGINT,
  zip STRING
) PARTITIONED BY (bucket(24, zip));
```

### Partition Evolution

```sql
-- Start with year partitioning
CREATE TABLE members (
  member_id BIGINT,
  registration_ts TIMESTAMP
) PARTITIONED BY (years(registration_ts));

-- Later, evolve to month partitioning
ALTER TABLE members ADD PARTITION FIELD months(registration_ts);

-- Drop old partition field
ALTER TABLE members DROP PARTITION FIELD years(registration_ts);

-- Replace derived column with transform
ALTER TABLE members 
REPLACE PARTITION FIELD registration_day 
WITH days(registration_ts) AS day_of_registration;
```

**Note:** Existing data keeps old partitioning, new data uses new scheme

### Copy-on-Write vs. Merge-on-Read

| Update Style | Read Speed | Write Speed | Best Practice |
|--------------|------------|-------------|---------------|
| **Copy-on-Write** | Fastest | Slowest | Read-heavy workloads |
| **Merge-on-Read (position deletes)** | Fast | Fast | Regular compaction |
| **Merge-on-Read (equality deletes)** | Slow | Fastest | Frequent compaction |

**Setting COW/MOR:**
```sql
-- At table creation
CREATE TABLE people (
  id INT,
  first_name STRING,
  last_name STRING
) TBLPROPERTIES (
  'write.delete.mode' = 'copy-on-write',
  'write.update.mode' = 'merge-on-read',
  'write.merge.mode' = 'merge-on-read'
);

-- After creation
ALTER TABLE people SET TBLPROPERTIES (
  'write.delete.mode' = 'merge-on-read'
);
```

### Metrics Collection

**Column Metrics Levels:**

| Level | Description |
|-------|-------------|
| `none` | No metrics |
| `counts` | Value counts, null counts only |
| `truncate(16)` | Counts + truncated min/max |
| `full` | Counts + full min/max |

**Setting Metrics:**
```sql
ALTER TABLE students SET TBLPROPERTIES (
  'write.metadata.metrics.column.name' = 'full',
  'write.metadata.metrics.column.age' = 'counts',
  'write.metadata.metrics.column.ssn' = 'none'
);
```

### Rewriting Manifests

```sql
CALL catalog.system.rewrite_manifests('MyTable');
CALL catalog.system.rewrite_manifests('MyTable', false); -- No Spark caching
```

### Expiring Snapshots

```sql
-- Expire snapshots older than 90 days, keep last 50
CALL catalog.system.expire_snapshots(
  'MyTable', 
  TIMESTAMP '2023-02-01 00:00:00.000', 
  100
);

-- Expire specific snapshot IDs
CALL catalog.system.expire_snapshots(
  table => 'MyTable',
  snapshot_ids => ARRAY(53)
);
```

### Removing Orphan Files

```sql
-- Dry run (preview)
CALL catalog.system.remove_orphan_files(
  table => 'MyTable',
  dry_run => true
);

-- Actually delete
CALL catalog.system.remove_orphan_files(
  table => 'MyTable',
  older_than => TIMESTAMP '2023-01-01'
);
```

### Write Distribution Mode

| Mode | Description | Best For |
|------|-------------|----------|
| `none` | No special distribution | Presorted data |
| `hash` | Hash-distribute by partition key | Balanced writes |
| `range` | Range-distribute by partition/sort | Clustered data |

```sql
ALTER TABLE employees 
SET TBLPROPERTIES (
  'write.distribution-mode' = 'hash',
  'write.merge.distribution-mode' = 'hash'
);
```

### Object Storage Optimization

```sql
-- Distribute files across prefixes to avoid throttling
ALTER TABLE MyTable SET TBLPROPERTIES (
  'write.object-storage.enabled' = true
);

-- Before: all files under same prefix
s3://bucket/table/field=value1/file1.parquet
s3://bucket/table/field=value1/file2.parquet

-- After: hash in path
s3://bucket/4809098/table/field=value1/file1.parquet
s3://bucket/5840329/table/field=value1/file2.parquet
```

### Bloom Filters

```sql
ALTER TABLE MyTable SET TBLPROPERTIES (
  'write.parquet.bloom-filter-enabled.column.col1' = true,
  'write.parquet.bloom-filter-max-bytes' = 1048576
);
```

---
