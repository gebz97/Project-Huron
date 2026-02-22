
## Chapter 7: Optimizing and Tuning Spark

### Viewing Spark Configurations

**From Spark application:**
```scala
import org.apache.spark.sql.SparkSession

def printConfigs(session: SparkSession): Unit = {
  val mconf = session.conf.getAll
  for ((k, v) <- mconf) {
    println(s"$k -> $v")
  }
}

// Set configs programmatically
val spark = SparkSession.builder
  .config("spark.sql.shuffle.partitions", 5)
  .config("spark.executor.memory", "2g")
  .master("local[*]")
  .appName("SparkConfig")
  .getOrCreate()
```

**From Spark shell:**
```scala
// Get all configs
spark.conf.getAll

// Check if config is modifiable
spark.conf.isModifiable("spark.sql.shuffle.partitions")

// Set config
spark.conf.set("spark.sql.shuffle.partitions", 5)

// Get config
spark.conf.get("spark.sql.shuffle.partitions")
```

**From command line:**
```bash
spark-submit \
  --conf spark.sql.shuffle.partitions=5 \
  --conf spark.executor.memory=2g \
  --class main.scala.chapter7.SparkConfig \
  jars/main-scala-chapter7_2.12-1.0.jar
```

### Dynamic Resource Allocation

```scala
// Enable dynamic allocation
spark.conf.set("spark.dynamicAllocation.enabled", "true")
spark.conf.set("spark.dynamicAllocation.minExecutors", "2")
spark.conf.set("spark.dynamicAllocation.maxExecutors", "20")
spark.conf.set("spark.dynamicAllocation.schedulerBacklogTimeout", "1m")
spark.conf.set("spark.dynamicAllocation.executorIdleTimeout", "2min")
```

### Executor Memory Layout

```
Executor Memory (spark.executor.memory)
┌────────────────────────────────────────┐
│ Reserved Memory (300 MB)               │  ← Safety against OOM
├────────────────────────────────────────┤
│ Execution Memory (60% of remaining)    │  ← Shuffles, joins, sorts
├────────────────────────────────────────┤
│ Storage Memory (40% of remaining)      │  ← Cached data
└────────────────────────────────────────┘
```

### Shuffle Tuning Configurations

| Configuration | Default | Recommendation |
|---------------|---------|----------------|
| `spark.shuffle.file.buffer` | 32 KB | 1 MB |
| `spark.file.transferTo` | true | false |
| `spark.shuffle.unsafe.file.output.buffer` | 32 KB | 1 MB |
| `spark.io.compression.lz4.blockSize` | 32 KB | 512 KB |
| `spark.shuffle.service.index.cache.size` | 100m | - |
| `spark.shuffle.registration.timeout` | 5000 ms | 120000 ms |
| `spark.shuffle.registration.maxAttempts` | 3 | 5 |

### Caching and Persistence

**cache() example:**
```scala
// Create DataFrame with 10M records
val df = spark.range(1 * 10000000).toDF("id")
            .withColumn("square", $"id" * $"id")

df.cache()        // Cache in memory
df.count()        // Materialize cache (takes 5.11 seconds)
df.count()        // Now get from cache (takes 0.44 seconds)
```

**persist() with StorageLevel:**
```scala
import org.apache.spark.storage.StorageLevel

// Persist to disk
df.persist(StorageLevel.DISK_ONLY)
df.count()  // Materialize (takes 2.08 seconds)
df.count()  // From disk (takes 0.38 seconds)

// Unpersist
df.unpersist()
```

**Storage Levels:**

| Level | Description |
|-------|-------------|
| `MEMORY_ONLY` | Store as objects in memory |
| `MEMORY_ONLY_SER` | Store as serialized objects in memory |
| `MEMORY_AND_DISK` | Memory, spill to disk if needed |
| `DISK_ONLY` | Store on disk only |
| `OFF_HEAP` | Store off-heap memory |
| `MEMORY_AND_DISK_SER` | Serialized in memory, spill to disk |

**Cache SQL Table:**
```scala
df.createOrReplaceTempView("dfTable")
spark.sql("CACHE TABLE dfTable")
spark.sql("SELECT count(*) FROM dfTable").show()
```

### Broadcast Hash Join (BHJ)

```
Small Table (broadcast)
       ↓
┌──────────────┐
│    Driver    │  ← Broadcast to all executors
└──────────────┘
    ↓     ↓     ↓
┌─────┐ ┌─────┐ ┌─────┐
│Exec1│ │Exec2│ │Exec3│
└─────┘ └─────┘ └─────┘
   ↓       ↓       ↓
Large Table partitions
```

```scala
import org.apache.spark.sql.functions.broadcast

// Force broadcast join
val joinedDF = playersDF.join(broadcast(clubsDF), "key1 === key2")

// Configure threshold (default 10 MB)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 10*1024*1024)

// Set to -1 to disable broadcast join
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

### Shuffle Sort Merge Join (SMJ)

**Default join for large tables:**
```scala
// Disable broadcast join to force SMJ
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")

val usersDF = (0 to 1000000).map(id => (id, s"user_$id", 
               s"user_$id@email.com")).toDF("uid", "login", "email")

val ordersDF = (0 to 1000000).map(r => (r, r, r.nextInt(10000)))
               .toDF("transaction_id", "quantity", "users_id")

val usersOrdersDF = ordersDF.join(usersDF, $"users_id" === $"uid")
```

**View physical plan:**
```scala
usersOrdersDF.explain()
// == Physical Plan ==
// *(3) SortMergeJoin [users_id#42], [uid#13], Inner
// :- *(1) Sort [users_id#42 ASC NULLS FIRST], false, 0
// :  +- Exchange hashpartitioning(users_id#42, 16)
// :     +- LocalTableScan ...
// +- *(2) Sort [uid#13 ASC NULLS FIRST], false, 0
//    +- Exchange hashpartitioning(uid#13, 16)
//       +- LocalTableScan ...
```

### Optimizing SMJ with Bucketing

**Create bucketed tables:**
```scala
import org.apache.spark.sql.SaveMode

// Bucket by join keys
usersDF.orderBy(asc("uid"))
  .write.format("parquet")
  .bucketBy(8, "uid")
  .mode(SaveMode.Overwrite)
  .saveAsTable("UsersTbl")

ordersDF.orderBy(asc("users_id"))
  .write.format("parquet")
  .bucketBy(8, "users_id")
  .mode(SaveMode.Overwrite)
  .saveAsTable("OrdersTbl")

// Cache tables
spark.sql("CACHE TABLE UsersTbl")
spark.sql("CACHE TABLE OrdersTbl")

// Read back and join
val usersBucketDF = spark.table("UsersTbl")
val ordersBucketDF = spark.table("OrdersTbl")
val joined = ordersBucketDF.join(usersBucketDF, $"users_id" === $"uid")
```

**After bucketing - no Exchange:**
```scala
joined.explain()
// == Physical Plan ==
// *(3) SortMergeJoin [users_id#165], [uid#62], Inner
// :- *(1) Sort [users_id#165 ASC NULLS FIRST], false, 0
// :  +- *(1) Filter isnotnull(users_id#165)
// :     +- Scan In-memory table `OrdersTbl`
// +- *(2) Sort [uid#62 ASC NULLS FIRST], false, 0
//    +- *(2) Filter isnotnull(uid#62)
//       +- Scan In-memory table `UsersTbl`
```

### Spark UI Tabs

**Jobs Tab:**
- Event Timeline
- List of completed jobs
- Duration metrics

**Stages Tab:**
- Stage DAG
- Task metrics (duration, GC time, shuffle bytes)
- Data skew indicators

**Executors Tab:**
- Memory usage
- Disk usage
- GC time
- Shuffle read/write

**Storage Tab:**
- Cached tables/DataFrames
- Memory distribution across partitions

**SQL Tab:**
- Query execution plans
- Physical operator metrics
- Scan, aggregate, exchange details

**Environment Tab:**
- Runtime properties
- Spark configs
- System properties
- Classpath

---
