
## Chapter 12: Spark 3.0 Features

### Dynamic Partition Pruning (DPP)

**Query:**
```sql
SELECT * FROM Sales JOIN Date ON Sales.date = Date.date
WHERE Date.year = 2019
```

**Without DPP:**
- Scan entire Sales table
- Join with filtered Date

**With DPP:**
```
1. Filter Date table (small) → build hash table
2. Broadcast hash table to executors
3. Executors probe hash table during Sales scan
4. Only read matching partitions from Sales
```

**Enabled by default in Spark 3.0**

### Adaptive Query Execution (AQE)

**Enable AQE:**
```sql
SET spark.sql.adaptive.enabled = true;
```

**Three Optimizations:**

1. **Coalesce shuffle partitions** - Reduce number of reducers
   ```
   spark.sql.adaptive.coalescePartitions.enabled = true
   ```

2. **Convert SMJ to BHJ** - If table small enough
   ```
   spark.sql.adaptive.skewJoin.enabled = true
   ```

3. **Handle skew joins** - Split skewed partitions

### SQL Join Hints

**Broadcast Hint:**
```sql
SELECT /*+ BROADCAST(a) */ id 
FROM a JOIN b ON a.key = b.key

SELECT /*+ BROADCAST(customers) */ * 
FROM customers, orders 
WHERE orders.custId = customers.custId
```

**Sort Merge Join Hint:**
```sql
SELECT /*+ MERGE(a, b) */ id 
FROM a JOIN b ON a.key = b.key
```

**Shuffle Hash Join Hint:**
```sql
SELECT /*+ SHUFFLE_HASH(a, b) */ id 
FROM a JOIN b ON a.key = b.key
```

**Shuffle-and-Replicate Nested Loop Hint:**
```sql
SELECT /*+ SHUFFLE_REPLICATE_NL(a, b) */ id 
FROM a JOIN b
```

### Catalog Plugin API

**Configure custom catalog:**
```properties
# In spark-defaults.conf
spark.sql.catalog.ndb_catalog com.ndb.ConnectorImpl
spark.sql.catalog.ndb_catalog.option1 value1
spark.sql.catalog.ndb_catalog.option2 value2
```

**Use custom catalog:**
```sql
SHOW TABLES ndb_catalog;
CREATE TABLE ndb_catalog.table_1;
SELECT * FROM ndb_catalog.table_1;
```

```python
df.writeTo("ndb_catalog.table_1")
df = spark.read.table("ndb_catalog.table_1")
```

### Accelerator-Aware Scheduler (GPU Support)

**Enable GPU discovery:**
```properties
# In spark-defaults.conf
spark.worker.resource.gpu.discoveryScript=/path/to/gpu-discovery.sh
spark.executor.resource.gpu.amount=2
spark.task.resource.gpu.amount=1
```

**Use GPUs in RDD:**
```scala
import org.apache.spark.BarrierTaskContext

val rdd = ...
rdd.barrier.mapPartitions { it =>
  val context = BarrierTaskContext.get()
  context.barrier()
  val gpus = context.resources().get("gpu").get.addresses
  // Launch external process that leverages GPU
  launchProcess(gpus)
}
```

### Structured Streaming UI Tab

**Enable:**
```properties
spark.sql.streaming.ui.enabled=true
spark.sql.streaming.ui.retainedProgressUpdates=100
spark.sql.streaming.ui.retainedQueries=100
```

**Shows:**
- Aggregate statistics of completed streaming jobs
- Input rate, process rate, input rows
- Batch duration, operation duration

### Redesigned Pandas UDFs

**Scalar Pandas UDF (Spark 3.0):**
```python
import pandas as pd
from pyspark.sql.functions import pandas_udf

# Use type hints instead of PandasUDFType
@pandas_udf("long")
def pandas_plus_one(v: pd.Series) -> pd.Series:
    return v + 1

df.withColumn("plus_one", pandas_plus_one("id")).show()
```

**Iterator Support (for model loading):**
```python
from typing import Iterator

@pandas_udf("long")
def predict(iterator: Iterator[pd.Series]) -> Iterator[pd.Series]:
    model = load_model()  # Load once
    for features in iterator:
        yield model.predict(features)
```

### New Pandas Function APIs

**mapInPandas():**
```python
def pandas_filter(iterator: Iterator[pd.DataFrame]) -> Iterator[pd.DataFrame]:
    for pdf in iterator:
        yield pdf[pdf.id == 1]

df.mapInPandas(pandas_filter, schema=df.schema).show()
```

**cogrouped map:**
```python
def asof_join(left: pd.DataFrame, right: pd.DataFrame) -> pd.DataFrame:
    return pd.merge_asof(left, right, on="time", by="id")

(df1.groupby("id").cogroup(df2.groupby("id"))
 .applyInPandas(asof_join, "time int, id int, v1 double, v2 string")
 .show())
```

### DataFrame.explain() Modes

```python
# Simple (default)
filtered.explain(mode="simple")

# Extended - physical and logical plans
filtered.explain(mode="extended")

# Codegen - generated Java code
filtered.explain(mode="codegen")

# Cost - optimizer cost estimates
filtered.explain(mode="cost")

# Formatted - readable output
filtered.explain(mode="formatted")
```

### SQL EXPLAIN FORMATTED

```sql
EXPLAIN FORMATTED 
SELECT * 
FROM tmp_spark_readme 
WHERE value like "%Spark%"

-- Output shows formatted physical plan with indentation
```

### GroupByKey Output Change

**Spark 2.x:**
```
+-----+--------+
|value|count(1)|
+-----+--------+
|   20|       1|
|    3|       3|
+-----+--------+
```

**Spark 3.0:**
```
+---+--------+
|key|count(1)|
+---+--------+
| 20|       1|
|  3|       3|
+---+--------+
```

**Preserve old behavior:**
```sql
SET spark.sql.legacy.dataset.nameNonStructGroupingKeyAsValue = true;
```

---
