
## Chapter 13: Metadata Management, Data Flow, and Lineage

### Metadata Management

**Metastore vs. Data Catalog:**
- **Metastore:** Service storing data about data (Hive Metastore)
- **Data Catalog:** Tool enabling users to locate high-quality data

### Hive Metastore

- Stores referential database and table data
- Location of data assets (path)
- Table type, partitions, columns, schema
- Accessed via SQL `SHOW` commands

**Limitation:** Only one catalog per Spark session

### Unity Catalog (OSS)

**Features:**
- Centralized metastore
- Three-tiered namespace: `{catalog}.{schema}.{table}`
- Unified governance for data and AI
- Managed/unmanaged volumes
- Interoperability with Iceberg, Parquet, CSV, JSON
- Apache 2.0 licensed

**Quickstart:**
```bash
git clone git@github.com:unitycatalog/unitycatalog.git
cd unitycatalog
bin/start-uc-server

# List tables
bin/uc table list --catalog unity --schema default
```

### Data Lineage

**Purpose:** Record movements, transformations, refinements from ingestion to destination

**Application Lineage:** Track jobs, versions, configurations
**Dataset Lineage:** Track sources → tables → outputs

### OpenLineage

**Core entities:**
- **Dataset** - Data source/sink
- **Job** - Process transforming data
- **Run** - Job execution

**Python client example:**
```python
from openlineage.client import OpenLineageClient, RunEvent, RunState
from uuid import uuid4

client = OpenLineageClient.from_environment()

job = Job(namespace='consumer', name='consumer.clickstream.orders')
run = Run(f"job:{str(uuid4())}")

# Start event
run_event = RunEvent(
    RunState.START,
    datetime.now().isoformat(),
    run, job, producer
)
client.emit(run_event)

# ... run application ...

# Complete event
run_event = RunEvent(
    RunState.COMPLETE,
    datetime.now().isoformat(),
    run, job, producer,
    inputs=[Dataset(namespace='consumer', name='consumer.clickstream')],
    outputs=[Dataset(namespace='consumer', name='consumer.orders')]
)
client.emit(run_event)
```

### Automating Data Life Cycles

**Table properties for retention:**
```sql
ALTER TABLE delta.`{table_path}`
SET TBLPROPERTIES (
  'catalog.table.gov.retention.enabled'='true',
  'catalog.table.gov.retention.date_col'='event_date',
  'catalog.table.gov.retention.policy'='interval 28 days'
);
```

**Convert interval string to IntervalType:**
```python
def convert_to_interval(interval: str):
    target = interval.lower().lstrip().replace("interval", "").lstrip()
    number, interval_type = re.split("\s+", target)
    amount = int(number)
    
    dt_interval = [None, None, None, None]
    if interval_type == "days":
        dt_interval[0] = lit(364 if amount > 365 else amount)
    elif interval_type == "hours":
        dt_interval[1] = lit(23 if amount > 24 else amount)
    # ... mins, secs
    
    return make_dt_interval(
        days=dt_interval[0],
        hours=dt_interval[1],
        mins=dt_interval[2],
        secs=dt_interval[3]
    )
```

**Calculate retention date:**
```python
props = DeltaTable.forPath(spark, table_path).detail().first()['properties']
enabled = bool(props.get('catalog.table.gov.retention.enabled', 'false'))
policy = props.get('catalog.table.gov.retention.policy', 'interval 90 days')

interval = convert_to_interval(policy)
rules = (spark.sql("select current_timestamp() as now")
         .withColumn("retain_after", to_date(col("now") - interval)))
```

### Table Properties for Monitoring

```sql
ALTER TABLE delta.`{table_path}`
SET TBLPROPERTIES (
  'catalog.table.deprecated'='false',
  'catalog.table.expectations.sla.refresh.frequency'='interval 1 hour',
  'catalog.table.expectations.checks.frequency'='interval 15 minutes',
  'catalog.table.expectations.checks.alert_after_num_failed'='3'
);
```

---
