
## Chapter 4: Diving into the Delta Lake Ecosystem

### Connectors Overview

**Delta Kernel** (Delta 3.0+):
- Common interface for all connectors
- Ensures correct operation
- Same behavior across connectors
- Quick implementation of new features

**UniForm** (Delta 3.0):
- Cross-table support for Delta, Iceberg, and Hudi

### Apache Flink Connector

**Maven dependency:**
```xml
<dependency>
  <groupId>io.delta</groupId>
  <artifactId>delta-flink</artifactId>
  <version>${delta-connectors-version}</version>
</dependency>
```

**DeltaSource API - Bounded Mode:**
```java
import org.apache.flink.core.fs.Path;
import org.apache.hadoop.conf.Configuration;

Path sourceTable = new Path("s3://bucket/delta/table_name");
Configuration hadoopConf = new Configuration();

DeltaSource<RowData> source = DeltaSource.forBoundedRowData(
  sourceTable, 
  hadoopConf
)
.columnNames("event_time", "event_type", "brand", "price")
.startingVersion(100L)
.option("parquetBatchSize", 5000)
.build();
```

**DeltaSource API - Continuous Mode:**
```java
DeltaSource<RowData> source = DeltaSource.forContinuousRowData(
  sourceTable, 
  hadoopConf
)
.updateCheckIntervalMills(60000L)
.ignoreDeletes(true)
.ignoreChanges(false)
.build();
```

**DeltaSink API:**
```java
Path deltaTable = new Path("s3://bucket/delta/table_name");
Configuration hadoopConf = new Configuration();
RowType rowType = getRowType(typeInfo);

FlinkSink.forRowData(input)
  .tableLoader(tableLoader)
  .overwrite(true)    // or upsert(true)
  .build();
```

**Kafka to Delta Example:**
```java
DataStreamSource<Ecommerce> stream = env.fromSource(
  kafkaSource,
  WatermarkStrategy.noWatermarks(),
  "kafka-source"
);

stream.map((MapFunction<Ecommerce, RowData>) Ecommerce::convertToRowData)
  .sinkTo(deltaSink)
  .name("delta-sink");
```

### Kafka Delta Ingest

**Install Rust:**
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**Build project:**
```bash
git clone git@github.com:delta-io/kafka-delta-ingest.git
cd kafka-delta-ingest

# Set up environment
docker compose up setup

# Build connector
cargo build
```

**Environment Variables:**

| Variable | Description | Default |
|----------|-------------|---------|
| `KAFKA_BROKERS` | Kafka broker string | localhost:9092 |
| `AWS_ENDPOINT_URL` | LocalStack testing | none |
| `AWS_ACCESS_KEY_ID` | Application identity | test |
| `AWS_SECRET_ACCESS_KEY` | Application authentication | test |

**Command-line Arguments:**

| Argument | Description | Example |
|----------|-------------|---------|
| `allowed_latency` | Buffer fill time | `--allowed_latency 60` |
| `app_id` | Application ID | `--app_id ingest-app` |
| `auto_offset_reset` | earliest or latest | `--auto_offset_reset earliest` |
| `checkpoints` | Record Kafka metadata | `--checkpoints` |
| `consumer_group_id` | Consumer group name | `--consumer_group_id ecomm` |
| `max_messages_per_batch` | Throttle messages | `--max_messages_per_batch 1600` |
| `min_bytes_per_file` | Minimum file size | `--min_bytes_per_file 64000000` |

**Run ingestion:**
```bash
cargo run ingest ecomm.v1.clickstream file:///dldg/ecomm-ingest/ \
  --allowed_latency 120 \
  --app_id clickstream_ecomm \
  --auto_offset_reset earliest \
  --checkpoints \
  --kafka 'localhost:9092' \
  --max_messages_per_batch 2000 \
  --transform 'date: substr(meta.producer.timestamp, 0, 10)' \
  --transform 'meta.kafka.offset: kafka.offset' \
  --transform 'meta.kafka.partition: kafka.partition' \
  --transform 'meta.kafka.topic: kafka.topic'
```

### Trino Connector

**Requirements:**
- Trino version > 373
- Network access to Delta Lake storage
- Access to Hive Metastore (HMS)
- Network access to HMS (port 9083)

**Docker Compose for Trino:**
```yaml
services:
  trinodb:
    image: trinodb/trino:426-arm64
    platform: linux/arm64
    volumes:
      - ./etc/catalog/delta.properties:/etc/trino/catalog/delta.properties
      - ./conf:/etc/hadoop/conf/
    ports:
      - "9090:8080"
    environment:
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
```

**Delta Lake Connector Properties (delta.properties):**
```properties
connector.name=delta_lake
hive.metastore=thrift
hive.metastore.uri=thrift://metastore:9083
delta.hive-catalog-name=metastore
delta.compression-codec=SNAPPY
delta.enable-non-concurrent-writes=true
delta.target-max-file-size=512MB
delta.unique-table-location=true
delta.vacuum.min-retention=7d
```

**Show Catalogs:**
```sql
trino> show catalogs;
 Catalog
---------
 delta
 system
 tpch
(3 rows)
```

**Create Schema:**
```sql
trino> create schema delta.bronze_schema;
CREATE SCHEMA
```

**Show Schemas:**
```sql
trino> show schemas from delta;
 Schema
------------
 default
 information_schema
 bronze_schema
(3 rows)
```

**Delta to Trino Type Mapping:**

| Delta | Trino |
|-------|-------|
| BOOLEAN | BOOLEAN |
| INTEGER | INTEGER |
| BYTE | TINYINT |
| SHORT | SMALLINT |
| LONG | BIGINT |
| FLOAT | REAL |
| DOUBLE | DOUBLE |
| DECIMAL(p,s) | DECIMAL(p,s) |
| STRING | VARCHAR |
| BINARY | VARBINARY |
| DATE | DATE |
| TIMESTAMP_NTZ | TIMESTAMP(6) |
| TIMESTAMP | TIMESTAMP(3) WITH TIME ZONE |
| ARRAY | ARRAY |
| MAP | MAP |
| STRUCT(...) | ROW(...) |

**CREATE TABLE Options:**

| Property | Description | Default |
|----------|-------------|---------|
| `location` | Filesystem URI | warehouse.dir |
| `partitioned_by` | Partition columns | No partitions |
| `checkpoint_interval` | Commit frequency | 10 (OSS), 100 (DBR) |
| `change_data_feed_enabled` | Track changes | false |
| `column_mapping_mode` | Parquet column mapping | none |

**Create Table:**
```sql
trino> use delta.bronze_schema;
CREATE TABLE ecomm_v1_clickstream (
  event_date DATE,
  event_time VARCHAR(255),
  event_type VARCHAR(255),
  product_id INTEGER,
  category_id BIGINT,
  category_code VARCHAR(255),
  brand VARCHAR(255),
  price DECIMAL(5,2),
  user_id INTEGER,
  user_session VARCHAR(255)
) WITH (
  partitioned_by = ARRAY['event_date'],
  checkpoint_interval = 30,
  change_data_feed_enabled = false,
  column_mapping_mode = 'name'
);
```

**Table Operations:**

```sql
-- List tables
show tables;

-- Describe table
describe delta.bronze_schema."ecomm_v1_clickstream";

-- Insert
INSERT INTO delta.bronze_schema."ecomm_v1_clickstream" VALUES
  (DATE '2023-10-01', '2023-10-01T19:10:05.704396Z', 'view', ...);

-- Update
UPDATE delta.bronze_schema."ecomm_v1_clickstream"
SET category_code = 'health.beauty.products'
WHERE category_id = 2103807459595387724;

-- Create table as select
CREATE TABLE delta.bronze_schema."ecomm_lite" AS
SELECT event_date, product_id, brand, price
FROM delta.bronze_schema."ecomm_v1_clickstream";

-- Vacuum
CALL delta.system.vacuum('bronze_schema', 'ecomm_v1_clickstream', '1d');

-- Optimize
ALTER TABLE delta.bronze_schema."ecomm_v1_clickstream" EXECUTE optimize;
ALTER TABLE delta.bronze_schema."ecomm_v1_clickstream" EXECUTE optimize WHERE event_date = '2023-10-01';

-- Table history
SELECT version, timestamp, operation 
FROM delta.bronze_schema."ecomm_v1_clickstream$history";

-- Change Data Feed
SELECT event_date, _change_type, _commit_version, _commit_timestamp
FROM TABLE(delta.system.table_changes(
  schema_name => 'bronze_schema',
  table_name => 'ecomm_v1_clickstream',
  since_version => 0
));

-- View table properties
SELECT * FROM delta.bronze_schema."ecomm_v1_clickstream$properties";

-- Modify properties
ALTER TABLE delta.bronze_schema."ecomm_v1_clickstream"
SET PROPERTIES "change_data_feed_enabled" = false;

-- Drop table
DROP TABLE delta.bronze_schema."ecomm_lite";
```

---
