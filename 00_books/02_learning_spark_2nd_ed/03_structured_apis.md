
## Chapter 3: Structured APIs

### What's Underneath an RDD?

**Three Vital Characteristics:**
1. **Dependencies** - How RDD is constructed
2. **Partitions** - Split work for parallelization
3. **Compute function** - `Partition => Iterator[T]`

**Problems with RDDs:**
- Opaque to Spark (can't optimize lambda expressions)
- No knowledge of data types
- Limited optimization

### RDD vs DataFrame Comparison

**RDD Approach (opaque):**
```python
# RDD - tells Spark HOW to compute
dataRDD = sc.parallelize([("Brooke", 20), ("Denny", 31), 
                          ("Jules", 30), ("TD", 35), ("Brooke", 25)])

agesRDD = (dataRDD
           .map(lambda x: (x[0], (x[1], 1)))
           .reduceByKey(lambda x, y: (x[0] + y[0], x[1] + y[1]))
           .map(lambda x: (x[0], x[1][0]/x[1][1])))
```

**DataFrame Approach (expressive):**
```python
# DataFrame - tells Spark WHAT to do
from pyspark.sql import SparkSession
from pyspark.sql.functions import avg

spark = SparkSession.builder.appName("AuthorsAges").getOrCreate()

data_df = spark.createDataFrame([("Brooke", 20), ("Denny", 31), 
                                  ("Jules", 30), ("TD", 35), ("Brooke", 25)],
                                 ["name", "age"])

avg_df = data_df.groupBy("name").agg(avg("age"))
avg_df.show()
```

### Spark Data Types

**Basic Scala Types:**
```scala
import org.apache.spark.sql.types._

val stringType = StringType
val intType = IntegerType
val longType = LongType
val doubleType = DoubleType
val booleanType = BooleanType
```

**Complex Types:**
```scala
// ArrayType
val arrayType = ArrayType(StringType, true)

// MapType
val mapType = MapType(StringType, IntegerType)

// StructType
val schema = StructType(Array(
  StructField("name", StringType, false),
  StructField("age", IntegerType, true),
  StructField("address", StructType(Array(
    StructField("street", StringType),
    StructField("city", StringType)
  )))
))
```

### Defining Schemas

**Two Ways:**

**1. Programmatic:**
```python
from pyspark.sql.types import *

schema = StructType([
    StructField("author", StringType(), False),
    StructField("title", StringType(), False),
    StructField("pages", IntegerType(), False)
])
```

**2. DDL String (simpler):**
```python
schema = "author STRING, title STRING, pages INT"
```

### Column Operations

```python
from pyspark.sql.functions import col, expr

# Select specific columns
blogs_df.select("Id", "First").show()

# Column arithmetic
blogs_df.select(expr("Hits * 2")).show(2)
blogs_df.select(col("Hits") * 2).show(2)

# Conditional column
blogs_df.withColumn(
    "Big Hitters", 
    expr("Hits > 10000")
).show()

# Concatenate columns
blogs_df.withColumn(
    "AuthorsId",
    concat(col("First"), col("Last"), col("Id"))
).select("AuthorsId").show(4)

# Sort
blogs_df.sort(col("Id").desc()).show()
blogs_df.sort($"Id".desc()).show()  # Scala
```

### Row Objects

```python
from pyspark.sql import Row

# Create a Row
blog_row = Row(6, "Reynold", "Xin", "https://tinyurl.6", 
               255568, "3/2/2015", ["twitter", "LinkedIn"])

# Access by index
blog_row[1]  # 'Reynold'

# Create DataFrame from Rows
rows = [Row("Matei Zaharia", "CA"), Row("Reynold Xin", "CA")]
authors_df = spark.createDataFrame(rows, ["Authors", "State"])
authors_df.show()
```

### DataFrameReader and DataFrameWriter

**Reading Data:**
```python
# CSV with schema
fire_df = (spark.read
           .format("csv")
           .option("header", "true")
           .schema(fire_schema)
           .load("/path/to/sf-fire-calls.csv"))

# JSON
json_df = spark.read.format("json").load("/path/to/file.json")

# Parquet (default)
parquet_df = spark.read.load("/path/to/file.parquet")
```

**Writing Data:**
```python
# Write as Parquet
fire_df.write.format("parquet").save("/path/to/output")

# Write as table
fire_df.write.format("parquet").saveAsTable("fire_calls_table")

# Write with mode
fire_df.write.mode("overwrite").save("/path/to/output")
```

### Common DataFrame Operations

**Projections and Filters:**
```python
# Select and filter
non_medical = (fire_df
               .select("IncidentNumber", "CallType")
               .where(col("CallType") != "Medical Incident"))
non_medical.show(5)

# Distinct values
fire_df.select("CallType").distinct().show()
```

**Aggregations:**
```python
# Count distinct call types
from pyspark.sql.functions import countDistinct

fire_df.agg(countDistinct("CallType").alias("DistinctCallTypes")).show()

# Group by and count
(fire_df
 .groupBy("CallType")
 .count()
 .orderBy("count", ascending=False)
 .show(10))
```

**Date/Time Conversions:**
```python
from pyspark.sql.functions import to_timestamp, year, month, day

# Convert string to timestamp
fire_ts_df = (fire_df
              .withColumn("IncidentDate", 
                          to_timestamp("CallDate", "MM/dd/yyyy"))
              .drop("CallDate"))

# Extract year
fire_ts_df.select(year("IncidentDate")).distinct().orderBy("year").show()
```

**Statistical Functions:**
```python
from pyspark.sql import functions as F

(fire_ts_df
 .select(F.sum("NumAlarms"), 
         F.avg("Delay"),
         F.min("Delay"),
         F.max("Delay"))
 .show())
```

### Dataset API

**Typed vs. Untyped:**

| Language | Main Abstraction | Type |
|----------|------------------|------|
| Scala | `Dataset[T]` and `DataFrame` (`Dataset[Row]`) | Both |
| Java | `Dataset<T>` | Typed |
| Python | `DataFrame` | Untyped |
| R | `DataFrame` | Untyped |

**Creating Datasets (Scala):**
```scala
// Define case class
case class DeviceIoTData(
  battery_level: Long, 
  c02_level: Long,
  cca2: String, 
  cca3: String, 
  cn: String,
  device_id: Long, 
  device_name: String, 
  humidity: Long,
  ip: String, 
  latitude: Double, 
  lcd: String,
  longitude: Double, 
  scale: String, 
  temp: Long,
  timestamp: Long
)

// Read and convert to Dataset
val ds = spark.read.json("/path/to/iot.json").as[DeviceIoTData]

// Dataset operations
val filtered = ds.filter(d => d.temp > 30 && d.humidity > 70)
filtered.show(5)
```

### Catalyst Optimizer

**Four Phases:**
1. **Analysis** - Resolve column/table names
2. **Logical Optimization** - Rule-based optimization
3. **Physical Planning** - Generate physical plans
4. **Code Generation** - Generate Java bytecode

```
SQL/DataFrame → AST → Analysis → Logical Plan → 
Optimized Logical Plan → Physical Plans → Cost Model → 
Selected Physical Plan → RDDs → Code Generation → Execution
```

**View Query Plan:**
```python
df.explain(True)  # Show detailed plan
```

---
