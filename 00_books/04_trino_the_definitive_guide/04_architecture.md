
## Chapter 4: Trino Architecture

### Coordinator and Workers

```
Client (CLI, JDBC, ODBC)
    ↓
┌──────────────┐
│ Coordinator  │ ← Handles queries, planning, scheduling
└──────────────┘
    ↓     ↓     ↓
┌─────┐ ┌─────┐ ┌─────┐
│Worker│ │Worker│ │Worker│ ← Execute tasks, process data
└─────┘ └─────┘ └─────┘
    ↓     ↓     ↓
Data Sources (HDFS, S3, RDBMS, NoSQL, ...)
```

**Components:**

| Component | Role |
|-----------|------|
| **Coordinator** | Receives SQL, parses, plans, schedules, manages workers |
| **Worker** | Executes tasks, processes data, communicates with data sources |
| **Discovery Service** | Nodes register and discover each other (runs on coordinator) |

### Connector-Based Architecture

```
Trino SPI (Service Provider Interface)
├── Metadata SPI (tables, columns, types)
├── Statistics SPI (row counts, table sizes)
├── Data Location SPI (splits, partitions)
└── Data Source SPI (actual data streaming)

Connectors implement SPI for each data source:
- Hive connector (HDFS, S3, ADLS, GCS)
- RDBMS connectors (PostgreSQL, MySQL, SQL Server)
- NoSQL connectors (Cassandra, MongoDB, Elasticsearch)
- Streaming connectors (Kafka, Kinesis)
- JMX, TPC-H, memory, blackhole connectors
```

### Query Execution Model

**Step-by-Step Query Processing:**

```
SQL Query
    ↓
┌──────────────┐
│    Parser    │ → Syntax check
└──────────────┘
    ↓
┌──────────────┐
│   Analyzer   │ → Resolve tables, columns, types
└──────────────┘
    ↓
┌──────────────┐
│   Planner    │ → Create logical plan
└──────────────┘
    ↓
┌──────────────┐
│  Optimizer   │ → Apply rules (predicate pushdown, join reorder)
└──────────────┘
    ↓
┌──────────────┐
│   Scheduler  │ → Create distributed plan with stages
└──────────────┘
    ↓
    Execution
```

**Key Concepts:**

| Term | Definition |
|------|------------|
| **Stage** | Runtime incarnation of plan fragment |
| **Task** | Unit of work within a stage |
| **Split** | Smallest unit of work (e.g., file segment) |
| **Driver** | Processes a split through operator pipeline |
| **Page** | Columnar data unit flowing between operators |
| **Operator** | Scan, filter, join, aggregate, etc. |

### Query Planning Example

**Example Query:**
```sql
SELECT (SELECT name FROM region r WHERE regionkey = n.regionkey) AS region_name,
       n.name AS nation_name,
       sum(totalprice) orders_sum
FROM nation n, orders o, customer c
WHERE n.nationkey = c.nationkey AND c.custkey = o.custkey
GROUP BY n.nationkey, regionkey, n.name
ORDER BY orders_sum DESC
LIMIT 5;
```

**Optimization Rules:**

| Rule | Description | Benefit |
|------|-------------|---------|
| **Predicate Pushdown** | Move filters closer to data source | Early data reduction |
| **Cross Join Elimination** | Reorder joins to avoid cross joins | Prevent explosive intermediate results |
| **TopN** | Combine ORDER BY + LIMIT | O(rows × log(limit)) vs O(rows × log(rows)) |
| **Partial Aggregation** | Pre-aggregate before joins | Reduce data flow |

### Cost-Based Optimizer (CBO)

**Cost Dimensions:**
- CPU time
- Memory usage
- Network bandwidth

**Join Strategies:**

| Strategy | Description | Best For |
|----------|-------------|----------|
| **Broadcast Join** | Small table copied to all workers | Small build side |
| **Distributed Join** | Both tables partitioned by join key | Large tables |

**Broadcast Join:**
```
Build side (small) → Broadcast to all workers
Probe side (large) → Partitioned across workers
Each worker has complete build side + part of probe side
```

**Distributed Join:**
```
Both tables partitioned by join key
Workers get matching partitions
No data duplication
```

**Table Statistics Required for CBO:**
- Number of rows
- Number of distinct values per column
- Fraction of NULL values
- Min/max values
- Average data size

**Collect Statistics:**
```sql
-- Trino ANALYZE
ANALYZE hive.ontime.flights;
ANALYZE hive.ontime.flights WITH (partitions = ARRAY[ARRAY['01-01-2019']]);

-- Enable stats on write
SET SESSION hive.collect-column-statistics-on-write = true;
```

**View Statistics:**
```sql
SHOW STATS FOR hive.ontime.flights;
SHOW STATS FOR (SELECT * FROM hive.ontime.flights WHERE year > 2010);
```

---
