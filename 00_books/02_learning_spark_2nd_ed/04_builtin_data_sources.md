
## Chapter 4: Built-in Data Sources

### Using Spark SQL in Applications

```python
from pyspark.sql import SparkSession

# Create SparkSession
spark = (SparkSession
         .builder
         .appName("SparkSQLExampleApp")
         .getOrCreate())

# Read CSV with schema inference
csv_file = "/databricks-datasets/learning-spark-v2/flights/departuredelays.csv"
df = (spark.read.format("csv")
      .option("inferSchema", "true")
      .option("header", "true")
      .load(csv_file))

# Create temporary view
df.createOrReplaceTempView("us_delay_flights_tbl")
```

### Basic SQL Queries

```sql
-- Find flights with distance > 1000 miles
SELECT distance, origin, destination 
FROM us_delay_flights_tbl 
WHERE distance > 1000 
ORDER BY distance DESC

-- Find delayed flights between SFO and ORD
SELECT date, delay, origin, destination 
FROM us_delay_flights_tbl 
WHERE delay > 120 AND origin = 'SFO' AND destination = 'ORD' 
ORDER BY delay DESC

-- CASE statement for labeling
SELECT delay, origin, destination,
  CASE
    WHEN delay > 360 THEN 'Very Long Delays'
    WHEN delay > 120 AND delay < 360 THEN 'Long Delays'
    WHEN delay > 60 AND delay < 120 THEN 'Short Delays'
    WHEN delay > 0 AND delay < 60 THEN 'Tolerable Delays'
    WHEN delay = 0 THEN 'No Delays'
    ELSE 'Early'
  END AS Flight_Delays
FROM us_delay_flights_tbl
ORDER BY origin, delay DESC
```

### SQL Tables and Views

**Create Database:**
```sql
CREATE DATABASE learn_spark_db;
USE learn_spark_db;
```

**Managed Table:**
```sql
-- SQL
CREATE TABLE managed_us_delay_flights_tbl (
  date STRING, delay INT, distance INT, 
  origin STRING, destination STRING
)
```

```python
# DataFrame API
flights_df.write.saveAsTable("managed_us_delay_flights_tbl")
```

**Unmanaged Table:**
```sql
CREATE TABLE us_delay_flights_tbl (
  date STRING, delay INT, distance INT, 
  origin STRING, destination STRING
)
USING csv
OPTIONS (PATH '/path/to/departuredelays.csv')
```

**Views:**
```sql
-- Temporary view
CREATE OR REPLACE TEMP VIEW us_origin_airport_JFK_tmp_view AS
SELECT date, delay, origin, destination 
FROM us_delay_flights_tbl 
WHERE origin = 'JFK'

-- Global temporary view
CREATE OR REPLACE GLOBAL TEMP VIEW us_origin_airport_SFO_global_tmp_view AS
SELECT date, delay, origin, destination 
FROM us_delay_flights_tbl 
WHERE origin = 'SFO'

-- Query global temp view (need global_temp prefix)
SELECT * FROM global_temp.us_origin_airport_SFO_global_tmp_view
```

### DataFrameReader Options

| Method | Arguments | Description |
|--------|-----------|-------------|
| `format()` | "parquet", "csv", "json", "jdbc", "orc", "avro" | File format |
| `option()` | ("mode", "PERMISSIVE"/"FAILFAST"/"DROPMALFORMED") | Error handling |
| | ("inferSchema", true/false) | Schema inference |
| | ("path", "path/to/file") | File path |
| `schema()` | DDL String or StructType | Explicit schema |
| `load()` | "/path/to/data" | Load data |

### DataFrameWriter Options

| Method | Arguments | Description |
|--------|-----------|-------------|
| `format()` | "parquet", "csv", etc. | Output format |
| `option()` | ("mode", "append"/"overwrite"/"ignore"/"error") | Save mode |
| `bucketBy()` | (numBuckets, col1, col2, ...) | Bucketing |
| `partitionBy()` | (col1, col2, ...) | Partitioning |
| `save()` | "/path/to/save" | Save to path |
| `saveAsTable()` | "table_name" | Save as table |

### Working with Parquet

**Read Parquet:**
```python
df = spark.read.format("parquet").load("/path/to/file.parquet")
# format("parquet") is optional (it's the default)
df = spark.read.load("/path/to/file.parquet")
```

**Write Parquet:**
```python
(df.write
 .format("parquet")
 .mode("overwrite")
 .option("compression", "snappy")
 .save("/tmp/data/parquet/df_parquet"))
```

### Working with JSON

**Read JSON:**
```python
# Single-line JSON (default)
df = spark.read.format("json").load("/path/to/file.json")

# Multi-line JSON
df = spark.read.format("json").option("multiline", "true").load("/path/to/file.json")
```

**Write JSON:**
```python
(df.write
 .format("json")
 .mode("overwrite")
 .option("compression", "snappy")
 .save("/tmp/data/json/df.json"))
```

**JSON Options:**

| Property | Values | Description |
|----------|--------|-------------|
| `compression` | none, gzip, snappy, etc. | Compression codec |
| `dateFormat` | yyyy-MM-dd | Date format |
| `multiline` | true/false | Multi-line mode |
| `allowUnquotedFieldNames` | true/false | Allow unquoted fields |

### Working with CSV

**Read CSV:**
```python
df = (spark.read.format("csv")
      .option("header", "true")
      .schema("col1 INT, col2 STRING, col3 DOUBLE")
      .option("mode", "FAILFAST")
      .load("/path/to/file.csv"))
```

**CSV Options:**

| Property | Values | Description |
|----------|--------|-------------|
| `sep` | any char | Field delimiter (default: ,) |
| `header` | true/false | First line as header |
| `inferSchema` | true/false | Infer column types |
| `mode` | PERMISSIVE/FAILFAST/DROPMALFORMED | Error handling |
| `dateFormat` | yyyy-MM-dd | Date format |
| `escape` | any char | Escape character |

### Working with Avro

**Read Avro:**
```python
df = spark.read.format("avro").load("/path/to/file.avro")
```

**Write Avro:**
```python
df.write.format("avro").mode("overwrite").save("/tmp/data/avro/df_avro")
```

**Avro Options:**

| Property | Default | Description |
|----------|---------|-------------|
| `avroSchema` | None | Avro schema in JSON |
| `recordName` | topLevelRecord | Record name |
| `recordNamespace` | "" | Record namespace |
| `compression` | snappy | Compression codec |

### Working with Images

```python
from pyspark.ml import image

image_dir = "/path/to/images/"
images_df = spark.read.format("image").load(image_dir)
images_df.printSchema()

# Schema:
# root
#  |-- image: struct
#  |    |-- origin: string
#  |    |-- height: integer
#  |    |-- width: integer
#  |    |-- nChannels: integer
#  |    |-- mode: integer
#  |    |-- data: binary
#  |-- label: integer
```

### Working with Binary Files (Spark 3.0)

```python
# Read all JPG files
binary_files_df = (spark.read.format("binaryFile")
                   .option("pathGlobFilter", "*.jpg")
                   .load("/path/to/images/"))

# Schema:
# - path: StringType
# - modificationTime: TimestampType
# - length: LongType
# - content: BinaryType
```

---
