
# 📖 Part I: Foundations

## Chapter 1: Introduction to Apache Spark

### The Genesis of Spark

**Google's Contributions (2003-2004):**
- Google File System (GFS) - Fault-tolerant distributed filesystem
- MapReduce (MR) - Parallel programming paradigm
- Bigtable - Scalable structured data storage

**Hadoop at Yahoo! (2006):**
- HDFS (Hadoop Distributed File System)
- MapReduce implementation
- **Shortcomings:**
  - Hard to manage and administer
  - Verbose API with boilerplate code
  - Performance overhead (disk I/O between map/reduce stages)
  - Not suitable for ML, streaming, or interactive SQL

```
Map Phase → Disk Write → Reduce Phase → Disk Write → Next Stage
```

### Spark's Early Years at AMPLab (2009)
- 10-20x faster than Hadoop MapReduce
- In-memory storage for intermediate results
- Unified APIs for multiple workloads

### What is Apache Spark?

**Four Key Characteristics:**

| Characteristic | Description |
|----------------|-------------|
| **Speed** | DAG scheduler, Tungsten engine, in-memory computation |
| **Ease of Use** | Simple APIs in Java, Scala, Python, R, SQL |
| **Modularity** | Unified libraries (SQL, Streaming, MLlib, GraphX) |
| **Extensibility** | Connectors to various data sources |

### Spark Components

```
┌─────────────────────────────────────┐
│           Spark SQL                 │
├─────────────────────────────────────┤
│        Spark Streaming              │
├─────────────────────────────────────┤
│           MLlib                     │
├─────────────────────────────────────┤
│           GraphX                    │
├─────────────────────────────────────┤
│      Spark Core Engine              │
└─────────────────────────────────────┘
```

### Spark Distributed Execution

**Components:**
- **Driver** - Orchestrates parallel operations
- **SparkSession** - Unified entry point (Spark 2.0+)
- **Cluster Manager** - Standalone, YARN, Mesos, Kubernetes
- **Executors** - Run tasks on worker nodes

**Deployment Modes:**

| Mode | Driver | Executor | Cluster Manager |
|------|--------|----------|-----------------|
| Local | Single JVM | Same JVM | Same host |
| Standalone | Any node | Each node | Any host |
| YARN (client) | Client (not cluster) | NodeManager's container | ResourceManager |
| YARN (cluster) | YARN Application Master | NodeManager's container | ResourceManager |
| Kubernetes | Pod | Pod | Kubernetes Master |

### Partitions and Parallelism

```
Data on Disk:
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Part 1│ │Part 2│ │Part 3│ │Part 4│
└──────┘ └──────┘ └──────┘ └──────┘
    ↓        ↓        ↓        ↓
Executor  Core 1   Core 2   Core 3   Core 4
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Task 1│ │Task 2│ │Task 3│ │Task 4│
└──────┘ └──────┘ └──────┘ └──────┘
```

**Code Example - Creating Partitions:**
```python
# Create DataFrame with 8 partitions
df = spark.range(0, 10000, 1, 8)
print(df.rdd.getNumPartitions())  # Output: 8

# Repartition existing DataFrame
log_df = spark.read.text("large_file.txt").repartition(8)
print(log_df.rdd.getNumPartitions())  # Output: 8
```

---
