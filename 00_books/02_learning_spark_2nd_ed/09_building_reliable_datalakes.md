
## Chapter 9: Building Reliable Data Lakes

### Storage Solutions Evolution

```
Databases (1970s-2000s)
    ↓
Data Lakes (2000s-2010s)
    ↓
Lakehouses (2018+)
```

### Databases

**Characteristics:**
- Structured data as tables
- Strict schema enforcement
- ACID transactions
- Tightly coupled storage and compute

**Limitations:**
- Expensive to scale out
- Poor support for non-SQL analytics
- Proprietary formats

### Data Lakes

**Characteristics:**
- Decoupled storage and compute
- Open file formats (Parquet, ORC, JSON)
- Commodity hardware
- Scalable horizontally

**Limitations:**
- No ACID guarantees
- No schema enforcement
- Inconsistent reads during writes
- Difficult data governance

### Lakehouses

**Characteristics:**
- ACID transactions
- Schema enforcement and evolution
- Open formats (Parquet)
- Support for diverse workloads (SQL, streaming, ML)
- Time travel and versioning

### Delta Lake

**Configure Delta Lake:**
```bash
# Start PySpark with Delta Lake
pyspark --packages io.delta:delta-core_2.12:0.7.0

# Or for Spark 2.4
pyspark --packages io.delta:delta-core_2.12:0.6.0
```

**Maven coordinates (Scala/Java):**
```xml
<dependency>
    <groupId>io.delta</groupId>
    <artifactId>delta-core_2.12</artifactId>
    <version>0.7.0</version>
</dependency>
```

### Loading Data into Delta Lake

**Convert Parquet to Delta:**
```python
# Read Parquet
sourcePath = "/databricks-datasets/learning-spark-v2/loans/loan-risks.snappy.parquet"
deltaPath = "/tmp/loans_delta"

# Write as Delta
(spark.read.format("parquet").load(sourcePath)
 .write.format("delta").save(deltaPath))

# Create view
spark.read.format("delta").load(deltaPath).createOrReplaceTempView("loans_delta")
```

### Streaming to Delta Lake

```python
newLoanStreamDF = ...  # Streaming DataFrame

checkpointDir = "/tmp/checkpoint"

streamingQuery = (newLoanStreamDF.writeStream
                  .format("delta")
                  .option("checkpointLocation", checkpointDir)
                  .trigger(processingTime="10 seconds")
                  .start(deltaPath))
```

### Schema Enforcement

**This will fail:**
```python
# Data with extra column 'closed'
loanUpdates = spark.createDataFrame([
    (1111111, 1000, 1000.0, "TX", False),
    (2222222, 2000, 0.0, "CA", True)
], ["loan_id", "funded_amnt", "paid_amnt", "addr_state", "closed"])

# This fails - schema mismatch
loanUpdates.write.format("delta").mode("append").save(deltaPath)
# Error: A schema mismatch detected when writing to the Delta table
```

**Schema evolution with mergeSchema:**
```python
# Allow schema merge
(loanUpdates.write.format("delta")
 .mode("append")
 .option("mergeSchema", "true")
 .save(deltaPath))
```

### UPDATE Operations

```python
from delta.tables import DeltaTable

deltaTable = DeltaTable.forPath(spark, deltaPath)

# Update all rows where addr_state = 'OR' to 'WA'
deltaTable.update("addr_state = 'OR'", {"addr_state": "'WA'"})
```

### DELETE Operations

```python
# Delete fully paid loans
deltaTable.delete("funded_amnt >= paid_amnt")
```

### MERGE (Upsert) Operations

```python
# Merge updates into table
(deltaTable
 .alias("t")
 .merge(loanUpdates.alias("s"), "t.loan_id = s.loan_id")
 .whenMatchedUpdateAll()
 .whenNotMatchedInsertAll()
 .execute())

# Insert-only merge (deduplication)
(deltaTable
 .alias("t")
 .merge(historicalUpdates.alias("s"), "t.loan_id = s.loan_id")
 .whenNotMatchedInsertAll()
 .execute())
```

### Operation History

```python
# Show last 3 operations
(deltaTable
 .history(3)
 .select("version", "timestamp", "operation", "operationParameters")
 .show(truncate=False))
```

### Time Travel

```python
# Query by timestamp
(spark.read
 .format("delta")
 .option("timestampAsOf", "2020-01-01")
 .load(deltaPath))

# Query by version
(spark.read
 .format("delta")
 .option("versionAsOf", "4")
 .load(deltaPath))
```

---
