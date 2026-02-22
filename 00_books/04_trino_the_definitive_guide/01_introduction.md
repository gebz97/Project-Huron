
## Chapter 1: Introducing Trino

### The Problems with Big Data

```
Challenge:
┌─────────────────────────────────────────────────────────────┐
│  Data everywhere: RDBMS, NoSQL, Object Storage, Streaming  │
│  Different query languages for each system                  │
│  Analysts know SQL, but data is locked in silos             │
│  Traditional data warehouses are expensive and slow         │
└─────────────────────────────────────────────────────────────┘
```

### Trino to the Rescue

**What is Trino?**
- Open source, distributed SQL query engine
- Designed for fast analytics on large datasets (gigabytes to petabytes)
- Queries data where it lives (no data migration required)
- Supports federated queries across multiple data sources

**Key Characteristics:**

| Feature | Description |
|---------|-------------|
| **Performance** | In-memory parallel processing, pipelined execution, bytecode generation |
| **Scale** | Horizontal scaling across cluster |
| **SQL-on-Anything** | Queries any data source with standard SQL |
| **Separation** | Storage and compute decoupled |

### Trino vs. Traditional Systems

```
Traditional Database:        Trino:
┌──────────────┐            ┌──────────────┐
│   Storage    │            │   Compute    │ ← Trino cluster
│   Compute    │ ← coupled  │   Engine     │
└──────────────┘            └──────────────┘
         ↓                          ↓
    Proprietary                 Open formats
     formats                   (Parquet, ORC)
                                   ↓
                            ┌──────────────┐
                            │   Storage    │ ← HDFS, S3, etc.
                            └──────────────┘
```

### Trino Use Cases

| Use Case | Description |
|----------|-------------|
| **Single SQL Access Point** | Query all databases from one place |
| **Data Warehouse Access** | Query warehouse AND source systems together |
| **SQL on Anything** | Query NoSQL, object storage, streaming systems |
| **Federated Queries** | Join data across different systems |
| **Virtual Data Warehouse** | Semantic layer without data movement |
| **Data Lake Query Engine** | SQL on HDFS, S3, ADLS, GCS |
| **ETL** | Transform and move data between systems |

### Trino Resources

```
Website:        https://trino.io
Documentation:  https://trino.io/docs
Community Chat: https://trinodb.slack.com
Source Code:    https://github.com/trinodb/trino
Book Repo:      https://github.com/trinodb/trino-the-definitive-guide
```

### Iris and Flight Data Sets

**Iris Data Set** - Famous ML classification dataset (150 records, 5 columns)

**Flight Data Set** - FAA flight data for complex queries and federated examples

---
