
## Chapter 5: External Data Sources

### User-Defined Functions (UDFs)

**Scala UDF:**
```scala
import org.apache.spark.sql.functions._

// Define function
val cubed = (s: Long) => s * s * s

// Register UDF
spark.udf.register("cubed", cubed)

// Use in SQL
spark.range(1, 9).createOrReplaceTempView("udf_test")
spark.sql("SELECT id, cubed(id) AS id_cubed FROM udf_test").show()
```

**Python UDF:**
```python
from pyspark.sql.types import LongType

# Define function
def cubed(s):
    return s * s * s

# Register UDF
spark.udf.register("cubed", cubed, LongType())

# Use in SQL
spark.range(1, 9).createOrReplaceTempView("udf_test")
spark.sql("SELECT id, cubed(id) AS id_cubed FROM udf_test").show()
```

### Pandas UDFs (Vectorized UDFs)

**Scalar Pandas UDF (Spark 3.0):**
```python
import pandas as pd
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import LongType

# Define function with type hints
def cubed(a: pd.Series) -> pd.Series:
    return a * a * a

# Create Pandas UDF
cubed_udf = pandas_udf(cubed, returnType=LongType())

# Apply to DataFrame
df = spark.range(1, 4)
df.select("id", cubed_udf("id")).show()
# +---+---+
# | id| id|
# +---+---+
# | 1 | 1 |
# | 2 | 8 |
# | 3 | 27|
# +---+---+
```

### Spark SQL Shell

```bash
# Start Spark SQL shell
./bin/spark-sql

# Create table
spark-sql> CREATE TABLE people (name STRING, age INT);

# Insert data
spark-sql> INSERT INTO people VALUES ("Michael", NULL);
spark-sql> INSERT INTO people VALUES ("Andy", 30);
spark-sql> INSERT INTO people VALUES ("Samantha", 19);

# Query
spark-sql> SELECT * FROM people WHERE age < 20;
# Samantha 19
```

### Beeline (JDBC Client)

```bash
# Start Thrift server
./sbin/start-thriftserver.sh

# Start Beeline
./bin/beeline

# Connect
!connect jdbc:hive2://localhost:10000

# Run queries
0: jdbc:hive2://localhost:10000> SHOW tables;
0: jdbc:hive2://localhost:10000> SELECT * FROM people;

# Stop Thrift server
./sbin/stop-thriftserver.sh
```

### JDBC Data Sources

**PostgreSQL:**
```python
# Read from PostgreSQL
jdbcDF = (spark.read
          .format("jdbc")
          .option("url", "jdbc:postgresql://[DBSERVER]")
          .option("dbtable", "[SCHEMA].[TABLENAME]")
          .option("user", "[USERNAME]")
          .option("password", "[PASSWORD]")
          .load())

# Write to PostgreSQL
(jdbcDF.write
 .format("jdbc")
 .option("url", "jdbc:postgresql://[DBSERVER]")
 .option("dbtable", "[SCHEMA].[TABLENAME]")
 .option("user", "[USERNAME]")
 .option("password", "[PASSWORD]")
 .save())
```

**MySQL:**
```python
# Read from MySQL
jdbcDF = (spark.read
          .format("jdbc")
          .option("url", "jdbc:mysql://[DBSERVER]:3306/[DATABASE]")
          .option("driver", "com.mysql.jdbc.Driver")
          .option("dbtable", "[TABLENAME]")
          .option("user", "[USERNAME]")
          .option("password", "[PASSWORD]")
          .load())
```

### Partitioning for JDBC

**Important Properties:**

| Property | Description |
|----------|-------------|
| `numPartitions` | Max number of partitions for parallelism |
| `partitionColumn` | Column to partition by (must be numeric, date, or timestamp) |
| `lowerBound` | Minimum value for partition stride |
| `upperBound` | Maximum value for partition stride |

**Example:**
```python
# Settings
# numPartitions: 10
# lowerBound: 1000
# upperBound: 10000
# Stride = 1000

# Creates these 10 queries:
# SELECT * FROM table WHERE partitionColumn BETWEEN 1000 and 2000
# SELECT * FROM table WHERE partitionColumn BETWEEN 2000 and 3000
# ...
# SELECT * FROM table WHERE partitionColumn BETWEEN 9000 and 10000
```

### Higher-Order Functions

**Sample Data:**
```python
from pyspark.sql.types import *

schema = StructType([StructField("celsius", ArrayType(IntegerType()))])
t_list = [[[35, 36, 32, 30, 40, 42, 38]], [[31, 32, 34, 55, 56]]]
t_c = spark.createDataFrame(t_list, schema)
t_c.createOrReplaceTempView("tc")
```

**transform() - Map function:**
```sql
SELECT celsius,
  transform(celsius, t -> ((t * 9) / 5) + 32) as fahrenheit 
FROM tc
```

**filter() - Filter array:**
```sql
SELECT celsius,
  filter(celsius, t -> t > 38) as high 
FROM tc
```

**exists() - Check condition:**
```sql
SELECT celsius,
  exists(celsius, t -> t = 38) as threshold
FROM tc
```

**reduce() - Reduce to single value:**
```sql
SELECT celsius,
  reduce(
    celsius, 
    0, 
    (t, acc) -> t + acc, 
    acc -> (acc / size(celsius) * 9 / 5) + 32
  ) as avgFahrenheit 
FROM tc
```

### Common DataFrame Operations

**Unions:**
```python
# Union two DataFrames
bar = departureDelays.union(foo)
```

**Joins:**
```python
# Inner join
foo.join(airports, airports.IATA == foo.origin) \
   .select("City", "State", "date", "delay", "distance", "destination") \
   .show()
```

**Windowing:**
```python
from pyspark.sql.window import Window
from pyspark.sql.functions import dense_rank

# For each origin, find top 3 destinations by delay
windowSpec = Window.partitionBy("origin").orderBy(col("TotalDelays").desc())

(departureDelaysWindow
 .select("origin", "destination", "TotalDelays",
         dense_rank().over(windowSpec).alias("rank"))
 .filter(col("rank") <= 3)
 .show())
```

**Modifications:**
```python
# Add column
foo2 = foo.withColumn("status", expr("CASE WHEN delay <= 10 THEN 'On-time' ELSE 'Delayed' END"))

# Drop column
foo3 = foo2.drop("delay")

# Rename column
foo4 = foo3.withColumnRenamed("status", "flight_status")
```

---
