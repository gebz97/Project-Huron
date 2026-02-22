
## Chapter 5: Maintaining Your Delta Lake

### Table Properties Reference

| Property | Data Type | Use With | Default |
|----------|-----------|----------|---------|
| `delta.logRetentionDuration` | CalendarInterval | Cleaning | interval 30 days |
| `delta.deletedFileRetentionDuration` | CalendarInterval | Cleaning | interval 1 week |
| `delta.setTransactionRetentionDuration` | CalendarInterval | Cleaning, Repairing | none |
| `delta.targetFileSize` | String | Tuning | none |
| `delta.tuneFileSizesForRewrites` | Boolean | Tuning | none |
| `delta.autoOptimize.optimizeWrite` | Boolean | Tuning | none |
| `delta.autoOptimize.autoCompact` | Boolean | Tuning | none |
| `delta.dataSkippingNumIndexedCols` | Int | Tuning | 32 |
| `delta.checkpoint.writeStatsAsStruct` | Boolean | Tuning | none |
| `delta.checkpoint.writeStatsAsJson` | Boolean | Tuning | true |
| `delta.randomizeFilePrefixes` | Boolean | Tuning | false |

### Custom Metadata Properties

| Property | Description |
|----------|-------------|
| `catalog.team_name` | Who is accountable for the table? |
| `catalog.engineering.comms.slack` | Slack permalink |
| `catalog.engineering.comms.email` | Engineering team email |
| `catalog.table.classification` | pii, sensitive-pii, general, all-access |

### Create Empty Table with Properties

```sql
CREATE TABLE IF NOT EXISTS default.covid_nyt (
  date DATE
) USING DELTA
TBLPROPERTIES('delta.logRetentionDuration'='interval 7 days');
```

### Populate Table

```python
from pyspark.sql.functions import to_date

(spark.read
 .format("parquet")
 .load("/opt/spark/work-dir/rs/data/COVID-19_NYT/*.parquet")
 .withColumn("date", to_date("date", "yyyy-MM-dd"))
 .write
 .format("delta")
 .mode("append")
 .saveAsTable("default.covid_nyt"))
```

### Schema Enforcement and Evolution

**Error if table exists:**
```python
# Throws AnalysisException (default mode = errorIfExists)
```

**Fix with append mode:**
```python
.mode("append")
```

**Schema mismatch error:**
```python
# Throws AnalysisException: schema mismatch
```

**Fix with mergeSchema:**
```python
.option("mergeSchema", "true")
```

### Manual Schema Evolution

```sql
ALTER TABLE default.covid_nyt ADD COLUMNS (
  county STRING,
  state STRING,
  fips INT,
  cases INT,
  deaths INT
);
```

```python
.option("mergeSchema", "false")
```

### Add/Modify Table Properties

```sql
ALTER TABLE default.covid_nyt
SET TBLPROPERTIES (
  'engineering.team_name'='dldg_authors',
  'engineering.slack'='delta-users.slack.com'
);
```

```python
from delta.tables import DeltaTable
dt = DeltaTable.forName(spark, 'default.covid_nyt')
dt.history(10).select("version", "timestamp", "operation").show()
```

### Remove Table Properties

```sql
ALTER TABLE default.covid_nyt UNSET TBLPROPERTIES('delta.logRetentionDuration');
```

### Small File Problem

**Create small files:**
```python
(DeltaTable.createIfNotExists(spark)
 .tableName("default.nonoptimal_covid_nyt")
 .addColumn("date", "DATE")
 .addColumn("county", "STRING")
 .addColumn("state", "STRING")
 .addColumn("fips", "INT")
 .addColumn("cases", "INT")
 .addColumn("deaths", "INT")
 .execute())

# Repartition to 9000 files
(spark.table("default.covid_nyt")
 .repartition(9000)
 .write
 .format("delta")
 .mode("overwrite")
 .saveAsTable("default.nonoptimal_covid_nyt"))
```

### OPTIMIZE Command

```sql
OPTIMIZE default.nonoptimal_covid_nyt;
```

```python
from delta.tables import DeltaTable
dt = DeltaTable.forName(spark, "default.nonoptimal_covid_nyt")
dt.optimize().executeCompaction()
```

**View optimization results:**
```python
dt.history().where(col("operation") == "OPTIMIZE").show()
```

### Z-Order Optimize

```sql
OPTIMIZE default.covid_nyt
ZORDER BY (state, county);
```

**Performance tuning properties:**
- `delta.dataSkippingNumIndexedCols` (int) - Number of stats columns (default 32)
- `delta.checkpoint.writeStatsAsStruct` (bool) - Enable columnar stats (default false)

### Partitioning Rules

1. **Don't partition by high cardinality columns** (e.g., userId with millions of values)
2. **Aim for ≥1 GB per partition**
3. **For tables <1 TB, consider no partitioning**

### Create Partitioned Table

```python
from pyspark.sql.types import DateType
from delta.tables import DeltaTable

(DeltaTable.createIfNotExists(spark)
 .tableName("default.covid_nyt_by_date")
 .addColumn("date", DateType(), nullable=False)
 .partitionedBy("date")
 .addColumn("county", "STRING")
 .addColumn("state", "STRING")
 .addColumn("fips", "INT")
 .addColumn("cases", "INT")
 .addColumn("deaths", "INT")
 .execute())
```

### Migrate to Partitioned Table

```python
(spark.table("default.covid_nyt")
 .write
 .format("delta")
 .mode("append")
 .option("mergeSchema", "false")
 .saveAsTable("default.covid_nyt_by_date"))
```

### View Partition Metadata

```python
(DeltaTable.forName(spark, "default.covid_nyt_by_date")
 .detail()
 .toJSON()
 .collect()[0])
```

### ReplaceWhere (Conditional Overwrite)

```python
recovery_table = spark.table("bronze.covid_nyt_by_date")
partition_col = "date"
partition_to_fix = "2021-02-17"
table_to_fix = "silver.covid_nyt_by_date"

(recovery_table
 .where(col(partition_col) == partition_to_fix)
 .write
 .format("delta")
 .mode("overwrite")
 .option("replaceWhere", f"{partition_col} = '{partition_to_fix}'")
 .saveAsTable("silver.covid_nyt_by_date"))
```

### Delete Partitions

```python
(DeltaTable
 .forName(spark, 'default.covid_nyt_by_date')
 .delete(col("date") < "2023-01-01"))
```

### Restore Table

```python
dt = DeltaTable.forName(spark, "silver.covid_nyt_by_date")
dt.history(10).select("version", "timestamp", "operation").show()
dt.restoreToVersion(0)  # Restore to version 0
```

### Vacuum

```python
dt.vacuum()  # Remove unreferenced files
```

**Retention properties:**
- `delta.logRetentionDuration` - History retention (default 30 days)
- `delta.deletedFileRetentionDuration` - Deleted file retention (default 1 week)

### Drop Table

```sql
DROP TABLE silver.covid_nyt_by_date;  -- Permanently deletes
```

---
