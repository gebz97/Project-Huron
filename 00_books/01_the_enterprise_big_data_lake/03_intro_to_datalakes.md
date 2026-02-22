
## Chapter 3: Introduction to Big Data and Data Science

### MapReduce (Google, 2004)
Breaks work into:
- **Mappers** - Run in parallel, process data chunks
- **Reducers** - Aggregate mapper outputs

### Hadoop
Open source MapReduce implementation

**HDFS (Hadoop File System):**
- Replicates blocks (default: 3 copies)
- Self-healing
- Large block size (64-128 MB)

**Schema on Read:**
- Apply schema when reading, not when writing
- Enables frictionless ingestion

### Key Hadoop Projects

| Category | On-Premises | Cloud (AWS/Azure/GCP) |
|----------|-------------|----------------------|
| Filesystem | HDFS, MapR-FS | S3, Blob, GCS |
| SQL Interface | Hive, Impala | RedShift, Athena |
| NoSQL | HBase | DynamoDB, Bigtable |
| Ingestion | Sqoop, Flume | Kinesis, Event Hub |
| Security | Ranger, Sentry | Security Center |

### Spark
- In-memory processing
- RDDs (Resilient Distributed Datasets)
- Faster than MapReduce
- Supports Scala, Java, Python, R

### Data Science

**The Three Pillars:**
1. Math/Statistics
2. Computer Science (Machine Learning)
3. Domain Knowledge

**Machine Learning Types:**
- **Supervised** - Trained on labeled data
- **Unsupervised** - Finds patterns without training

**Key Challenges:**
- **Explainability** - Can you explain why the model decided X?
- **Model Drift** - Models become less accurate over time
- **Data Drift** - Input data characteristics change

---
