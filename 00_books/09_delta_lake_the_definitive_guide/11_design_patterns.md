
## Chapter 11: Successful Design Patterns

### Comcast Smart Remote

**Scale:**
- 14 billion voice interactions (2018-2019)
- Up to 15 million transactions/second
- Petabytes of data

**Before Delta Lake:**
- 32 concurrent jobs
- 640 virtual machines
- Complex sessionization

**After Delta Lake:**
- Single Spark job
- 64 virtual machines
- 10x reduction in compute

**Additional benefits:**
- MLflow integration for reproducible ML
- 70% faster queries than Redshift (pilot)

### Scribd - Delta Rust

**Problem:** Many small Kafka streams, expensive Spark clusters

**Solution:** `delta-rs` + `kafka-delta-ingest`

**Cost savings:**
- 70-90 pipelines in production
- 100x cheaper than equivalent Spark applications
- Serverless via AWS Fargate

**Metrics monitored:** Deserialization logs, failures, Arrow batches, file sizes, lag

### DoorDash - CDC and Flink

**CDC Framework:**
- 450 streams (24/7)
- 1000+ EC2 nodes
- 800 GB ingested daily from Kafka
- 80 TB daily processing
- <30 minute latency

**Requirements met:**
- <1 day latency
- Lakehouse design
- Schema evolution
- Data backfilling
- Open source

**Flink Integration:**
- Petabytes of customer events
- ACID guarantees at scale
- No write locks
- Compaction while streaming

---
