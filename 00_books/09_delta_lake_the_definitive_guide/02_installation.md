
## Chapter 2: Installing Delta Lake

### Delta Lake Docker Image

**Pre-installed components:**
- Apache Arrow
- DataFusion (Rust query engine)
- ROAPI (read-only APIs)
- Rust
- Python, PySpark, Scala shells
- JupyterLab

**Run Docker container:**
```bash
# With bash entrypoint
docker run --name delta_quickstart --rm -it \
  --entrypoint bash delta_quickstart

# With JupyterLab
docker run --name delta_quickstart --rm -it \
  -p 8888-8889:8888-8889 delta_quickstart
```

### Delta Lake for Python

```python
import pandas as pd
from deltalake.writer import write_deltalake
from deltalake import DeltaTable

# Create Pandas DataFrame
df = pd.DataFrame(range(5), columns=["id"])

# Write Delta Lake table
write_deltalake("/tmp/deltars_table", df)

# Append new data
df2 = pd.DataFrame(range(6, 11), columns=["id"])
write_deltalake("/tmp/deltars_table", df2, mode="append")

# Read and show
dt = DeltaTable("/tmp/deltars_table")
dt.to_pandas()
```

**Verify files:**
```bash
ls -lsga /tmp/deltars_table
# 0-f3c05c4277a2-0.parquet
# 1-674ccf40faae-0.parquet
# _delta_log/
```

### PySpark Shell

```bash
$SPARK_HOME/bin/pyspark --packages io.delta:delta-core_2.12:2.3.0 \
  --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
  --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog"
```

```python
# Create DataFrame
data = spark.range(0, 5)

# Write to Delta
data.write.format("delta").save("/tmp/delta-table")

# Read from Delta
df = spark.read.format("delta").load("/tmp/delta-table")
df.show()
```

### Scala Shell

```bash
$SPARK_HOME/bin/spark-shell --packages io.delta:delta-core_2.12:2.3.0 \
  --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
  --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog"
```

```scala
val data = spark.range(0, 5)
data.write.format("delta").save("/tmp/delta-table")
val df = spark.read.format("delta").load("/tmp/delta-table")
df.show()
```

### Delta Rust API

```bash
# Read metadata and files
cd rs
cargo run --example read_delta_table

# Query with DataFusion
cargo run --example read_delta_datafusion
```

### ROAPI (Read-Only APIs)

```bash
# Start ROAPI
nohup roapi --addr-http 0.0.0.0:8080 \
  --table 'deltars_table=/tmp/deltars_table/,format=delta' \
  --table 'covid19_nyt=/opt/spark/work-dir/rs/data/COVID-19_NYT,format=delta' &

# Query
curl -X POST -d "SELECT * FROM deltars_table" localhost:8080/api/sql
```

### Native Delta Lake Libraries

**Python bindings (delta-rs):**
```bash
pip install deltalake
```

```python
from deltalake import DeltaTable
dt = DeltaTable("/tmp/deltars_table")
dt.to_pandas()
```

### Apache Spark with Delta Lake

**Prerequisite:** Java 8, 11, or 17

**Spark SQL Shell:**
```bash
bin/spark-sql --packages io.delta:delta-core_2.12:2.3.0 \
  --conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
  --conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog"
```

```sql
CREATE TABLE delta.`/tmp/delta-table` USING DELTA 
AS SELECT col1 AS id FROM VALUES 0, 1, 2, 3, 4;

SELECT * FROM delta.`/tmp/delta-table`;
```

### PySpark Declarative API

```bash
pip install delta-spark
```

```python
from delta import *

builder = pyspark.sql.SparkSession.builder.appName("MyApp") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")

spark = configure_spark_with_delta_pip(builder).getOrCreate()
```

### Databricks Community Edition

**Create Cluster:**
1. Compute → Create Compute
2. Select Databricks Runtime 13.3 LTS
3. Name cluster (e.g., "Delta_Lake_DLDG")
4. Create Cluster

**Import Notebook:**
1. Workspace → ⋮ (three dots) → Import
2. URL → Paste notebook URL → Import
3. Attach to cluster

---
