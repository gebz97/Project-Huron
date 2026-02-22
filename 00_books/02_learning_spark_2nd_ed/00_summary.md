# Learning Spark, 2nd Edition - Summary

## 📚 Book Overview

**Authors:** Jules S. Damji, Brooke Wenig, Tathagata Das, Denny Lee (all from Databricks)
**Publisher:** O'Reilly Media
**Focus:** Lightning-fast data analytics with Apache Spark 3.0

This book provides a comprehensive guide to Apache Spark, covering its evolution through Spark 2.x and Spark 3.0, with practical examples in Scala, Python, and SQL.

---

# 📊 Summary Tables

## Key Spark Configurations

| Configuration | Default | Description |
|---------------|---------|-------------|
| `spark.sql.shuffle.partitions` | 200 | Number of shuffle partitions |
| `spark.sql.autoBroadcastJoinThreshold` | 10MB | Max size for broadcast join |
| `spark.sql.adaptive.enabled` | false | Enable AQE (Spark 3.0) |
| `spark.dynamicAllocation.enabled` | false | Enable dynamic resource allocation |
| `spark.executor.memory` | 1g | Executor memory |
| `spark.sql.files.maxPartitionBytes` | 128MB | Max partition size |

## DataFrameReader Formats

| Format | Read | Write |
|--------|------|-------|
| Parquet | `spark.read.parquet()` | `df.write.parquet()` |
| JSON | `spark.read.json()` | `df.write.json()` |
| CSV | `spark.read.csv()` | `df.write.csv()` |
| Avro | `spark.read.format("avro")` | `df.write.format("avro")` |
| ORC | `spark.read.orc()` | `df.write.orc()` |
| Image | `spark.read.format("image")` | Not supported |
| Binary | `spark.read.format("binaryFile")` | Not supported |

## Storage Levels

| Level | Space | CPU | Where | Serialized |
|-------|-------|-----|-------|------------|
| `MEMORY_ONLY` | High | Low | Memory | No |
| `MEMORY_ONLY_SER` | Low | High | Memory | Yes |
| `MEMORY_AND_DISK` | High | Medium | Memory + Disk | No |
| `MEMORY_AND_DISK_SER` | Low | High | Memory + Disk | Yes |
| `DISK_ONLY` | Low | High | Disk | Yes |
| `OFF_HEAP` | Low | Medium | Off-heap | Yes |

## MLlib Algorithms

| Algorithm | Type | Package |
|-----------|------|---------|
| Linear Regression | Regression | `pyspark.ml.regression.LinearRegression` |
| Logistic Regression | Classification | `pyspark.ml.classification.LogisticRegression` |
| Decision Tree | Both | `pyspark.ml.regression.DecisionTreeRegressor` |
| Random Forest | Both | `pyspark.ml.regression.RandomForestRegressor` |
| Gradient Boosted Trees | Both | `pyspark.ml.regression.GBTRegressor` |
| K-Means | Clustering | `pyspark.ml.clustering.KMeans` |
| PCA | Dimension Reduction | `pyspark.ml.feature.PCA` |

---

*"Simple things should be simple, complex things should be possible." - Alan Kay*