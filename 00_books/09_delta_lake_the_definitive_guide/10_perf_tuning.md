
## Chapter 10: Performance Tuning

### Performance Objectives

| Objective | Focus | Techniques |
|-----------|-------|------------|
| **Read Performance** | Data consumers | Partitioning, indexing, statistics |
| **Write Performance** | Data producers | Optimized writes, autocompaction |

### Query Patterns

| Pattern | Description | Optimization |
|---------|-------------|--------------|
| **Point queries** | Single record | Indexes, smaller files |
| **Range queries** | Set of records | Partitioning, statistics |
| **Aggregations** | Summary by group | Partitioning, Z-ordering |

### Partitioning

**Structure:**
```
table/
├── membership_type=free/
│   └── part-00000-...parquet
└── membership_type=paid/
    └── part-00001-...parquet
```

**Rules:**
- ≥1 GB per partition
- No partitioning for tables <1 TB
- Avoid high cardinality columns

**Create partitioned table:**
```python
write_deltalake(
  "/tmp/delta/partitioning.example.delta",
  data=df,
  mode="overwrite",
  partition_by=["membership_type"]
)
```

### File Statistics

**Example stats from transaction log:**
```json
{
  "numRecords": 2,
  "minValues": {"id": 2, "name": "Customer 2"},
  "maxValues": {"id": 4, "name": "Customer 4"},
  "nullCount": {"id": 0, "name": 0}
}
```

**Benefits:**
- `SELECT max(id)` from stats (no data scan)
- `SELECT count(*)` from stats (no data scan)
- File skipping for filtered queries

### OPTIMIZE

```sql
OPTIMIZE delta.`/path/to/table`
```

**Behavior:**
- Lists all active files
- Combines small files
- Target size ~1 GB
- Heavy I/O operation

**With ZORDER:**
```sql
OPTIMIZE delta.`/path/to/table`
ZORDER BY (column1, column2)
```

### Z-Ordering

**Concept:** Space-filling curve for multi-dimensional clustering

```
Linear sorting (1D):
File A: x=1..3, y=random
File B: x=4..6, y=random
File C: x=7..9, y=random

Z-ordering (2D):
File A: (x=1..3, y=1..3) or (x=4..6, y=1..3) or ...
File B: clusters based on both dimensions
```

**Benefits:**
- Fewer files to read for multi-column filters
- File sizes may vary (prefer clustering over uniform size)

### Autocompaction (Databricks)

```sql
SET spark.databricks.delta.autoCompact.enabled = true;
SET spark.databricks.delta.autoCompact.maxFileSize = 128MB;
SET spark.databricks.delta.autoCompact.minNumFiles = 50;
```

### Optimized Writes

```sql
SET spark.databricks.delta.optimizeWrites.enabled = true;
```

**Before:** Multiple executors → multiple files per partition
**After:** Shuffle before write → fewer, optimized files

### Statistics Tuning

```sql
-- Limit indexed columns
ALTER TABLE delta.`example`
SET TBLPROPERTIES("delta.dataSkippingNumIndexedCols"=5);

-- Reorder columns
ALTER TABLE delta.`example` CHANGE articleDate FIRST;
ALTER TABLE delta.`example` CHANGE textCol AFTER revisionTimestamp;
```

### CLUSTER BY (Liquid Clustering)

**Create table with clustering:**
```sql
CREATE TABLE example.wikipages
CLUSTER BY (id)
AS (SELECT *, date(revisionTimestamp) AS articleDate FROM source_view);
```

**Modify clustering:**
```sql
ALTER TABLE example.wikipages CLUSTER BY (articleDate);
ALTER TABLE example.wikipages CLUSTER BY NONE;
```

**Requirements:**
- Writer version ≥ 7
- Reader version ≥ 3
- Declare at table creation (cannot add later)

**Benefits:**
- No partitioning needed
- Change clustering keys anytime
- Row-level concurrency
- Eager clustering during writes

### Bloom Filter Index

**Create index:**
```python
from pyspark.sql.functions import countDistinct

cdf = spark.table("example.wikipages")
raw_items = cdf.agg(countDistinct(cdf.id)).collect()[0][0]
num_items = int(raw_items * 1.25)  # Add 25% padding

spark.sql(f"""
CREATE BLOOMFILTER INDEX ON TABLE example.wikipages
FOR COLUMNS (id OPTIONS (fpp=0.05, numItems={num_items}))
""")
```

**Configuration:**
- `fpp` (false positive probability) - Lower = more accurate, larger index
- `maxExpectedFpp` - If calculated fpp exceeds threshold, index not written

**Supported types:** Numeric, datetime, strings, bytes (not nested)

---
