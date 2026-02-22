
## Chapter 7: Streaming In and Out of Your Delta Lake

### Streaming vs. Batch Processing

**Key difference: Latency**

```
Batch Processing:
[File 1] [File 2] [File 3] [File 4] [File 5] [File 6]
    ↓        ↓        ↓        ↓        ↓        ↓
        Process 1         |         Process 2
        (Files 1-3)       |        (Files 4-6)

Stream Processing:
[File 1] → Process → Output
[File 2] → Process → Output
[File 3] → Process → Output (continuous)
```

### Streaming Terminology

| Term | Description |
|------|-------------|
| **Source** | Unbounded data source (Kafka, Kinesis, files) |
| **Sink** | Output destination (storage, message queue) |
| **Checkpoint** | Tracks progress for failure recovery |
| **Watermark** | Limit on how late data can be accepted |

### Delta as Source

**Spark Structured Streaming:**
```python
streamingDeltaDf = (spark
  .readStream
  .format("delta")
  .option("ignoreDeletes", "true")
  .load("/files/delta/user_events"))
```

**Flink:**
```java
DeltaSource<RowData> source = DeltaSource.forContinuousRowData(
  sourceTable,
  hadoopConf
).build();
```

### Delta as Sink

**Spark:**
```python
(streamingDeltaDf
 .writeStream
 .format("delta")
 .outputMode("append")
 .start("/<delta_path>"))
```

**End-to-end pipeline:**
```python
(spark
 .readStream
 .format("delta")
 .load("/files/delta/user_events")
 .writeStream
 .format("delta")
 .outputMode("append")
 .start("/<delta_path>"))
```

### Streaming Options

**Limit Input Rate:**

| Option | Description | Default |
|--------|-------------|---------|
| `maxFilesPerTrigger` | Max new files per microbatch | 1000 |
| `maxBytesPerTrigger` | Approximate data per microbatch | (soft max) |

**⚠️ Warning:** If `logRetentionDuration` < trigger interval, older transactions can be skipped

### Ignore Updates/Deletes

**ignoreDeletes:**
- Ignores delete operations (no new files created)
- Data must be partitioned by delete column
- Downstream needs separate delete handling

```python
streamingDeltaDf = (spark
  .readStream
  .format("delta")
  .option("ignoreDeletes", "true")
  .load("/files/delta/user_events"))
```

**ignoreChanges:**
- New files from updates/deletes come through as new
- May cause duplicates downstream

```python
streamingDeltaDf = (spark
  .readStream
  .format("delta")
  .option("ignoreChanges", "true")
  .load("/files/delta/user_events"))
```

### Initial Processing Position

**By Version:**
```python
(spark
 .readStream
 .format("delta")
 .option("startingVersion", "5")
 .load("/files/delta/user_events"))
```

**By Timestamp:**
```python
(spark
 .readstream
 .format("delta")
 .option("startingTimestamp", "2023-04-18")
 .load("/files/delta/user_events"))
```

### Initial Snapshot with Event Time Order

```python
(spark
 .readstream
 .format("delta")
 .option("withEventTimeOrder", "true")
 .load("/files/delta/user_events")
 .withWatermark("event_time", "10 seconds"))
```

### Idempotent Stream Writes (Spark)

**Problem:** `foreachBatch` to multiple destinations can cause partial failures

**Solution:** Use `txnAppId` and `txnVersion`

```python
app_id = "unique-app-id"

def writeToDeltaLakeTableIdempotent(batch_df, batch_id):
    # Location 1
    (batch_df
     .write
     .format("delta")
     .option("txnVersion", batch_id)
     .option("txnAppId", app_id)
     .save("/<delta_path_1>"))
    
    # Location 2
    (batch_df
     .write
     .format("delta")
     .option("txnVersion", batch_id)
     .option("txnAppId", app_id)
     .save("/<delta_path_2>"))

(sourceDf
 .writeStream
 .foreachBatch(writeToDeltaLakeTableIdempotent)
 .start())
```

### Merge in Streaming

```python
from delta.tables import *

def upsertToDelta(microBatchDf, batchId):
    deltaTable = DeltaTable.forName(spark, "retail_db.transactions_silver")
    
    (deltaTable.alias("dt")
     .merge(
       source=microBatchDf.alias("sdf"),
       condition="sdf.t_id = dt.t_id"
     )
     .whenMatchedDelete(condition="sdf.operation = 'DELETE'")
     .whenMatchedUpdate(set={
       "t_id": "sdf.t_id",
       "transaction_date": "sdf.transaction_date",
       "item_count": "sdf.item_count",
       "amount": "sdf.amount"
     })
     .whenNotMatchedInsert(values={
       "t_id": "sdf.t_id",
       "transaction_date": "sdf.transaction_date",
       "item_count": "sdf.item_count",
       "amount": "sdf.amount"
     })
     .execute())

(changesStream
 .writeStream
 .foreachBatch(upsertToDelta)
 .outputMode("update")
 .start())
```

### Performance Metrics

**Spark Structured Streaming metrics:**
- `numInputRows` - Rows processed in last microbatch
- `inputRowsPerSecond` - Input rate
- `processedRowsPerSecond` - Processing rate

**Backpressure metrics (Databricks):**
- `numBytesOutstanding` - Outstanding work in bytes
- `numFilesOutstanding` - Outstanding work in files

### Change Data Feed (CDF)

**Enable on new table:**
```sql
CREATE TABLE student (id INT, name STRING, age INT)
TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

**Enable on existing table:**
```sql
ALTER TABLE myDeltaTable SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

**Read CDF (batch):**
```python
(spark.read.format("delta")
 .option("readChangeFeed", "true")
 .option("startingVersion", 0)
 .option("endingVersion", 10)
 .table("myDeltaTable"))

(spark.read.format("delta")
 .option("readChangeFeed", "true")
 .option("startingTimestamp", '2023-04-01 05:45:46')
 .option("endingTimestamp", '2023-04-21 12:00:00')
 .table("myDeltaTable"))
```

**Read CDF (streaming):**
```python
(spark.readStream.format("delta")
 .option("readChangeFeed", "true")
 .option("startingVersion", 0)
 .load("/pathToMyDeltaTable"))
```

**CDF Schema:**

| Column | Type | Description |
|--------|------|-------------|
| `_change_type` | string | insert, update_preimage, update_postimage, delete |
| `_commit_version` | long | Transaction log version |
| `_commit_timestamp` | timestamp | Commit time |

---
