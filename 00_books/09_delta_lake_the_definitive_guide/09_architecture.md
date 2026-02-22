
## Chapter 9: Architecting Your Lakehouse

### Lakehouse Architecture

**Definition:** Open data management architecture combining data lake flexibility with data warehouse ACID transactions.

```
Dual-Tier Architecture:
Siloed Data → Data Lake → Data Warehouse
               (raw)      (cleansed)

Lakehouse Architecture:
Siloed Data → Lakehouse (single platform)
```

### Dual-Tier Architecture Flow

1. Extract operational data → landing zones (`/raw/*`)
2. Read, clean, transform → `/cleansed`
3. Join, normalize → Data Warehouse

**Problems:**
- Multiple copies of data
- Two sources of truth = both may be out of sync
- Increased complexity and cost

### Lakehouse Benefits

- **Transaction support** - ACID guarantees
- **Schema enforcement** - Data integrity, audit log
- **BI support** - SQL, JDBC interfaces
- **Separation of storage and compute** - Scale independently
- **Open standards** - Parquet, Delta protocol
- **End-to-end streaming** - Unified batch/streaming
- **Diverse workloads** - SQL to deep learning

### Foundations with Delta Lake

**Open File Format:** Apache Parquet
**Self-describing metadata:** No separate metastore needed
**Open table specification:** No vendor lock-in

### UniForm (Universal Format)

**Enable Iceberg support:**
```sql
-- At creation
CREATE TABLE T(c1 INT) USING DELTA
TBLPROPERTIES('delta.universalFormat.enabledFormats' = 'iceberg, hudi');

-- After creation
ALTER TABLE T SET TBLPROPERTIES(
  'delta.columnMapping.mode' = 'name',
  'delta.enableIcebergCompatV2' = 'true',
  'delta.universalFormat.enabledFormats' = 'iceberg'
);
```

**Spark requirements:**
```bash
-- Iceberg
--packages io.delta:io.delta:delta-iceberg_2.12:<version>

-- Hudi
--packages io.delta:io.delta:delta-hudi_2.12:<version>
```

### Transaction Support

**Serializable writes:** Concurrent writers using optimistic concurrency
**Snapshot isolation:** Readers see consistent snapshot
**Incremental processing:** Read only changes since last version
**Time travel:** View table state at any point

**Traditional batch job state:**
```bash
./run-some-batch-job.py \
  --startTime x \
  --recordsPerBatch 10000 \
  --lastRecordId z
```

**With Delta:**
```bash
./run-some-batch-job.py --startingVersion 10 --recordsPerBatch 10000
```

### Schema-on-Write vs. Schema-on-Read

| Approach | Description | Delta Lake |
|----------|-------------|------------|
| **Schema-on-write** | Check schema before write | Yes - protects data quality |
| **Schema-on-read** | Apply schema when reading | No - can lead to data swamps |

### Medallion Architecture

```
Bronze (raw) → Silver (cleansed) → Gold (curated)
   ↓                 ↓                  ↓
Raw data        Filtered, cleaned    Business KPIs
as-is           Augmented data       Aggregates
```

**Bronze Layer (Minimal transformations):**
```python
bronze_layer_stream = (
  spark.readStream
  .format("kafka")
  .load()
  .select(col("key"), col("value"), col("topic"), col("timestamp"))
  .withColumn("event_date", to_date(col("timestamp")))
  .writeStream
  .format("delta")
  .partitionBy("event_date")
  .toTable("bronze.ecomm_raw")
)
```

**Permissive passthrough (block corrupt data):**
```python
from pyspark.sql.types import StructType, StructField, StringType

known_schema = (StructType.fromJson(...)
 .add(StructField('_corrupt', StringType(), True)))

happy_df = (
  spark.read.options(**{
    "inferSchema": "false",
    "mode": "PERMISSIVE",
    "columnNameOfCorruptRecord": "_corrupt"
  })
  .schema(known_schema)
  .json(...)
)
```

**Silver Layer (cleansed):**
```python
def transform_from_json(input_df):
    return input_df.withColumn("ecomm",
      from_json(
        col("value").cast(StringType()),
        known_schema,
        options={'mode': 'PERMISSIVE', 'columnNameOfCorruptRecord': '_corrupt'}
      ))

def transform_for_silver(input_df):
    return (input_df
      .select(
        col("event_date").alias("ingest_date"),
        col("timestamp").alias("ingest_timestamp"),
        col("ecomm.*")
      )
      .where(col("_corrupt").isNull())
      .drop("_corrupt"))

medallion_stream = (
  delta_source.readStream
  .transform(transform_from_json)
  .transform(transform_for_silver)
  .writeStream
  .option('mergeSchema', 'false')
  .toTable(f"{managed_silver_table}"))
```

**Gold Layer (curated):**
```python
top5 = (
  silver_table
  .groupBy("ingest_date", "category_id")
  .agg(
    count(col("product_id")).alias("impressions"),
    min(col("price")).alias("min_price"),
    avg(col("price")).alias("avg_price"),
    max(col("price")).alias("max_price")
  )
  .orderBy(desc("impressions"))
  .limit(5))

(top5
 .write
 .format("delta")
 .mode("overwrite")
 .saveAsTable(f"gold.{topN_products_daily}"))
```

### Streaming Medallion Architecture

```
Kafka → Bronze (stream) → Silver (stream) → Gold (streaming/views)
         append            incremental        periodic
```

---
