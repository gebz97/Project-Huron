
## Chapter 8: Structured Streaming

### Evolution of Stream Processing

**Traditional (record-at-a-time):**
```
Source → Node 1 → Node 2 → Node 3 → Sink
        (1 rec)   (1 rec)   (1 rec)
```

**Spark Micro-batch:**
```
Stream → [Batch] → [Batch] → [Batch] → Sink
          t1 sec   t2 sec   t3 sec
```

### Programming Model

**Key Concept: Stream as Unbounded Table**

```
Input Stream          Result Table
┌─────────────┐      ┌─────────────┐
│ record 1    │  →   │ result 1    │
│ record 2    │  →   │ result 2    │
│ record 3    │  →   │ result 3    │
│ ...         │      │ ...         │
└─────────────┘      └─────────────┘
     (unbounded)           ↓
                      Output Sink
```

### Output Modes

| Mode | Description | Supported By |
|------|-------------|--------------|
| **Append** | Only new rows since last trigger | Stateless queries only |
| **Update** | Only updated rows since last trigger | Most queries |
| **Complete** | Entire result table | Aggregations only |

### Five Steps to Define a Streaming Query

**1. Define Input Source:**
```python
lines = (spark
         .readStream
         .format("socket")
         .option("host", "localhost")
         .option("port", 9999)
         .load())
```

**2. Transform Data:**
```python
from pyspark.sql.functions import split, col

words = lines.select(split(col("value"), "\\s").alias("word"))
counts = words.groupBy("word").count()
```

**3. Define Output Sink and Mode:**
```python
writer = counts.writeStream.format("console").outputMode("complete")
```

**4. Specify Processing Details:**
```python
checkpointDir = "/tmp/checkpoint"
writer2 = (writer
           .trigger(processingTime="1 second")
           .option("checkpointLocation", checkpointDir))
```

**5. Start the Query:**
```python
streamingQuery = writer2.start()
streamingQuery.awaitTermination()
```

### Complete Streaming Word Count Example

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import split, col

spark = SparkSession.builder.appName("StreamingWordCount").getOrCreate()

# Read stream from socket
lines = (spark
         .readStream
         .format("socket")
         .option("host", "localhost")
         .option("port", 9999)
         .load())

# Transform
words = lines.select(split(col("value"), "\\s").alias("word"))
counts = words.groupBy("word").count()

# Write to console
checkpointDir = "/tmp/wordcount-checkpoint"
streamingQuery = (counts
                  .writeStream
                  .format("console")
                  .outputMode("complete")
                  .trigger(processingTime="1 second")
                  .option("checkpointLocation", checkpointDir)
                  .start())

streamingQuery.awaitTermination()
```

**Run the example:**
```bash
# Terminal 1: Start netcat server
nc -lk 9999

# Terminal 2: Run the Spark streaming job
./bin/spark-submit streaming_wordcount.py
```

### Checkpointing for Fault Tolerance

**Exactly-once guarantees require:**
1. **Replayable source** - Can reread data range
2. **Deterministic computations** - Same input → same output
3. **Idempotent sink** - Handles duplicate writes

**Restart a failed query:**
```python
# Just run the same code with same checkpoint location
# It will resume from where it left off
```

### Monitoring Streaming Queries

**lastProgress():**
```python
streamingQuery.lastProgress()
# {
#   "id": "ce011fdc-8762-4dcb-84eb-a77333e28109",
#   "runId": "88e2ff94-ede0-45a8-b687-6316fbef529a",
#   "numInputRows": 10,
#   "inputRowsPerSecond": 120.0,
#   "processedRowsPerSecond": 200.0,
#   ...
# }
```

**status():**
```python
streamingQuery.status()
# {
#   "message": "Waiting for data to arrive",
#   "isDataAvailable": false,
#   "isTriggerActive": false
# }
```

### File Source

```python
# Read from directory
inputDir = "/path/to/json/files"
fileSchema = "key INT, value INT"

inputDF = (spark
           .readStream
           .format("json")
           .schema(fileSchema)
           .load(inputDir))

# Write to directory
outputDir = "/path/to/output"
checkpointDir = "/tmp/checkpoint"

streamingQuery = (resultDF
                  .writeStream
                  .format("parquet")
                  .option("path", outputDir)
                  .option("checkpointLocation", checkpointDir)
                  .start())
```

### Kafka Source

**Read from Kafka:**
```python
inputDF = (spark
           .readStream
           .format("kafka")
           .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
           .option("subscribe", "events")
           .load())

# Schema of Kafka DataFrame:
# - key: binary
# - value: binary
# - topic: string
# - partition: int
# - offset: long
# - timestamp: long
# - timestampType: int
```

**Write to Kafka:**
```python
# Result DataFrame must have:
# - key (optional): string or binary
# - value (required): string or binary
# - topic (optional): string

streamingQuery = (counts
                  .selectExpr("cast(word as string) as key",
                              "cast(count as string) as value")
                  .writeStream
                  .format("kafka")
                  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
                  .option("topic", "wordCounts")
                  .outputMode("update")
                  .option("checkpointLocation", checkpointDir)
                  .start())
```

### foreachBatch() - Custom Sink

```python
def writeToCassandra(updatedCountsDF, batchId):
    # Write using batch data source
    (updatedCountsDF
     .write
     .format("org.apache.spark.sql.cassandra")
     .mode("append")
     .options(table="counts", keyspace="test")
     .save())

streamingQuery = (counts
                  .writeStream
                  .foreachBatch(writeToCassandra)
                  .outputMode("update")
                  .option("checkpointLocation", checkpointDir)
                  .start())
```

### Watermarks for Late Data

```python
from pyspark.sql.functions import window

# Set watermark (10 minutes)
(sensorReadings
 .withWatermark("eventTime", "10 minutes")
 .groupBy("sensorId", window("eventTime", "10 minutes", "5 minutes"))
 .mean("value"))
```

### Streaming Joins

**Stream-Static Join:**
```python
# Static DataFrame
impressionsStatic = spark.read.parquet("/path/to/impressions")

# Streaming DataFrame
clicksStream = spark.readStream.parquet("/path/to/clicks")

# Join
matched = clicksStream.join(impressionsStatic, "adId")
```

**Stream-Stream Inner Join with Watermarks:**
```python
# Define watermarks
impressionsWithWatermark = (impressions
                            .selectExpr("adId AS impressionAdId", "impressionTime")
                            .withWatermark("impressionTime", "2 hours"))

clicksWithWatermark = (clicks
                       .selectExpr("adId AS clickAdId", "clickTime")
                       .withWatermark("clickTime", "3 hours"))

# Join with time constraint
(impressionsWithWatermark
 .join(clicksWithWatermark,
       expr("""clickAdId = impressionAdId AND 
               clickTime BETWEEN impressionTime AND 
               impressionTime + interval 1 hour""")))
```

### mapGroupsWithState() - Arbitrary State

```scala
// Define state case class
case class UserStatus(userId: String, active: Boolean)

// Define state update function
def updateUserStatus(
    userId: String,
    newActions: Iterator[UserAction],
    state: GroupState[UserStatus]
): UserStatus = {
  
  val userStatus = state.getOption.getOrElse(UserStatus(userId, false))
  
  newActions.foreach { action => 
    userStatus.updateWith(action) 
  }
  
  state.update(userStatus)
  userStatus
}

// Apply to stream
val latestStatuses = userActions
  .groupByKey(userAction => userAction.userId)
  .mapGroupsWithState(updateUserStatus)
```

---
