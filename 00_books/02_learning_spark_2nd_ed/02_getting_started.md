
## Chapter 2: Getting Started

### Step 1: Downloading Apache Spark

```bash
# Download Spark (pre-built for Hadoop 2.7)
wget https://archive.apache.org/dist/spark/spark-3.0.0/spark-3.0.0-bin-hadoop2.7.tgz

# Extract
tar -xf spark-3.0.0-bin-hadoop2.7.tgz
cd spark-3.0.0-bin-hadoop2.7

# List contents
ls
# LICENSE  README.md  RELEASE  R  bin  conf  data  examples  jars  kubernetes  python  sbin  yarn
```

**Alternative - Install PySpark via pip:**
```bash
# Install PySpark only (for Python users)
pip install pyspark

# Install with extras
pip install pyspark[sql,ml,mllib]
```

**Set JAVA_HOME:**
```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

### Step 2: Using Spark Shells

**PySpark Shell:**
```bash
./bin/pyspark
```

```python
>>> spark.version
'3.0.0'
>>> strings = spark.read.text("README.md")
>>> strings.show(5, truncate=False)
+--------------------------------------------------------------------------------+
|value                                                                           |
+--------------------------------------------------------------------------------+
|# Apache Spark                                                                  |
|                                                                                |
|Spark is a unified analytics engine for large-scale data processing. It        |
|provides high-level APIs in Scala, Java, Python, and R, and an optimized       |
|engine that supports general computation graphs for data analysis. It also     |
|supports a rich set of higher-level tools including Spark SQL for SQL and      |
|DataFrames, MLlib for machine learning, GraphX for graph processing,           |
|and Structured Streaming for stream processing.                                |
+--------------------------------------------------------------------------------+
>>> strings.count()
109
```

**Scala Spark Shell:**
```bash
./bin/spark-shell
```

```scala
scala> spark.version
res0: String = 3.0.0

scala> val strings = spark.read.text("README.md")
strings: org.apache.spark.sql.DataFrame = [value: string]

scala> strings.show(5, false)
...

scala> strings.count()
res2: Long = 109
```

**Exit Spark Shell:**
```bash
# Press Ctrl-D
```

### Spark Application Concepts

```
Spark Application
       ↓
┌──────────────┐
│   Driver     │  ← Creates SparkSession
└──────────────┘
       ↓
┌──────────────┐
│     Job      │  ← Triggered by action (count(), save(), collect())
└──────────────┘
       ↓
┌──────────────┐
│    Stage     │  ← Boundaries where shuffle occurs
└──────────────┘
       ↓
┌──────────────┐
│    Tasks     │  ← One per partition, executed by executors
└──────────────┘
```

### Transformations and Actions

| Transformations (Lazy) | Actions (Eager) |
|------------------------|-----------------|
| `select()` | `show()` |
| `filter()` | `count()` |
| `groupBy()` | `collect()` |
| `join()` | `save()` |
| `orderBy()` | `take()` |

**Example:**
```python
# Transformations (nothing happens yet)
strings = spark.read.text("README.md")
filtered = strings.filter(strings.value.contains("Spark"))

# Action (triggers execution)
count = filtered.count()  # Returns 20
```

### Narrow vs. Wide Transformations

```
Narrow (no shuffle):          Wide (shuffle required):
┌────┐    ┌────┐             ┌────┐    ┌────┐
│Part│───>│Part│             │Part│───>│    │
│ 1  │    │ 1' │             │ 1  │    │    │
└────┘    └────┘             └────┘    │    │
                                      │Part│
┌────┐    ┌────┐             ┌────┐    │ 1' │
│Part│───>│Part│             │Part│───>│    │
│ 2  │    │ 2' │             │ 2  │    │    │
└────┘    └────┘             └────┘    └────┘
   filter(), map()              groupBy(), join()
```

### Spark UI

Access at: `http://localhost:4040`

**Tabs:**
- Jobs - List of all jobs
- Stages - Stage details and DAG
- Storage - Cached DataFrames
- Environment - Configuration
- Executors - Resource usage
- SQL - SQL query plans

### First Standalone Application

**M&M Counting Example - Python (mnmcount.py):**

```python
import sys
from pyspark.sql import SparkSession
from pyspark.sql.functions import count

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: mnmcount <file>", file=sys.stderr)
        sys.exit(-1)
    
    # Build SparkSession
    spark = (SparkSession
             .builder
             .appName("PythonMnMCount")
             .getOrCreate())
    
    # Read data
    mnm_file = sys.argv[1]
    mnm_df = (spark.read.format("csv")
              .option("header", "true")
              .option("inferSchema", "true")
              .load(mnm_file))
    
    # Aggregate counts
    count_mnm_df = (mnm_df
                    .select("State", "Color", "Count")
                    .groupBy("State", "Color")
                    .agg(count("Count").alias("Total"))
                    .orderBy("Total", ascending=False))
    
    # Show results
    count_mnm_df.show(n=60, truncate=False)
    print("Total Rows = %d" % count_mnm_df.count())
    
    # Filter for California
    ca_count_mnm_df = (mnm_df
                       .select("State", "Color", "Count")
                       .where(mnm_df.State == "CA")
                       .groupBy("State", "Color")
                       .agg(count("Count").alias("Total"))
                       .orderBy("Total", ascending=False))
    
    ca_count_mnm_df.show(n=10, truncate=False)
    spark.stop()
```

**Run the application:**
```bash
# Set log level to WARN (optional)
cp conf/log4j.properties.template conf/log4j.properties
# Edit conf/log4j.properties: set log4j.rootCategory=WARN

# Submit the job
$SPARK_HOME/bin/spark-submit mnmcount.py data/mnm_dataset.csv
```

**Sample Output:**
```
+-----+------+-----+
|State|Color |Total|
+-----+------+-----+
|CA   |Yellow|1807 |
|WA   |Green |1779 |
|OR   |Orange|1743 |
...
+-----+------+-----+

For California:
+-----+------+-----+
|State|Color |Total|
+-----+------+-----+
|CA   |Yellow|1807 |
|CA   |Green |1723 |
|CA   |Brown |1718 |
+-----+------+-----+
```

**Scala Version (build.sbt):**
```scala
name := "main/scala/chapter2"
version := "1.0"
scalaVersion := "2.12.10"

libraryDependencies ++ Seq(
  "org.apache.spark" %% "spark-core" % "3.0.0-preview2",
  "org.apache.spark" %% "spark-sql" % "3.0.0-preview2"
)
```

**Build and run Scala version:**
```bash
sbt clean package
$SPARK_HOME/bin/spark-submit --class main.scala.chapter2.MnMcount \
    jars/main-scala-chapter2_2.12-1.0.jar data/mnm_dataset.csv
```

---
