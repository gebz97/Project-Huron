
## Chapter 6: Connectors

### Configuration

**Catalog Properties File (etc/catalog/name.properties):**
```properties
connector.name=<connector-name>
# connector-specific properties
```

### RDBMS Connector Example: PostgreSQL

**Catalog file (etc/catalog/postgresql.properties):**
```properties
connector.name=postgresql
connection-url=jdbc:postgresql://db.example.com:5432/database
connection-user=root
connection-password=secret
```

**Usage:**
```sql
SHOW CATALOGS;
SHOW SCHEMAS IN postgresql;
USE postgresql.airline;
SHOW TABLES;
SELECT code, name FROM airport WHERE code = 'ORD';
```

**Query Pushdown:**
```
Trino Query:
SELECT state, count(*) FROM airport WHERE country = 'US' GROUP BY state;

Pushed to PostgreSQL:
SELECT state FROM airport WHERE country = 'US';
```

**Multiple JDBC Connections for Joins:**
```
Query joining two PostgreSQL tables:
┌─────────────┐
│   Trino     │
└─────────────┘
    ↓       ↓
JDBC Conn  JDBC Conn
    ↓       ↓
PostgreSQL  PostgreSQL
 Table A     Table B
```

### Other RDBMS Connectors

**MySQL:**
```properties
connector.name=mysql
connection-url=jdbc:mysql://example.net:3306
connection-user=root
connection-password=secret
```

**AWS Redshift:**
```properties
connector.name=redshift
connection-url=jdbc:postgresql://example.net:5439/database
connection-user=root
connection-password=secret
```

**Microsoft SQL Server:**
```properties
connector.name=sqlserver
connection-url=jdbc:sqlserver://example.net:1433;databaseName=sales
connection-user=root
connection-password=secret
```

### TPC-H and TPC-DS Connectors

**Catalog file (etc/catalog/tpch.properties):**
```properties
connector.name=tpch
```

**Available Schemas:**
```sql
SHOW SCHEMAS FROM tpch;
-- information_schema, sf1, sf100, sf1000, sf10000, tiny
```

**Table Sizes:**

| Schema | Orders Row Count |
|--------|------------------|
| tiny | 15,000 |
| sf1 | 1,500,000 |
| sf3 | 4,500,000 |
| sf100 | 150,000,000 |

### Hive Connector

**Hadoop/Hive Concepts:**
- **HDFS** - Distributed file system
- **Hive Metastore (HMS)** - Stores table metadata
- **Hive table format** - Directory-based table organization

**Catalog file (etc/catalog/hive.properties):**
```properties
connector.name=hive-hadoop2
hive.metastore.uri=thrift://example.net:9083
```

**Create Schema:**
```sql
-- HDFS
CREATE SCHEMA hive.web WITH (location = 'hdfs://starburst-oreilly/web');

-- S3
CREATE SCHEMA hive.web WITH (location = 's3://example-org/web');
```

**Managed vs. External Tables:**

| Table Type | Data Ownership | DROP Behavior |
|------------|----------------|---------------|
| **Managed** | Hive/Trino | Deletes metadata AND data |
| **External** | External system | Deletes metadata ONLY |

**Create Managed Table:**
```sql
CREATE TABLE hive.web.page_views (
  view_time timestamp,
  user_id bigint,
  page_url varchar,
  view_date date,
  country varchar
);
```

**Create External Table:**
```sql
CREATE TABLE hive.web.page_views (
  view_time timestamp,
  user_id bigint,
  page_url varchar,
  view_date date,
  country varchar
) WITH (
  external_location = 's3://example-org/page_views'
);
```

**Partitioned Table:**
```sql
CREATE TABLE hive.web.page_views (
  view_time timestamp,
  user_id bigint,
  page_url varchar,
  view_date date,
  country varchar
) WITH (
  partitioned_by = ARRAY['view_date', 'country']
);
```

**Dynamic Partitioning:**
```sql
-- Insert into partitioned table
INSERT INTO page_views_ext SELECT * FROM page_views;

-- Sync partitions
CALL system.sync_partition_metadata('web', 'page_views', 'FULL');

-- Drop partition
DELETE FROM hive.web.page_views WHERE view_date = DATE '2019-01-14';
```

**File Formats:**
```sql
CREATE TABLE hive.web.page_views (
  view_time timestamp,
  user_id bigint,
  page_url varchar,
  view_date date,
  country varchar
) WITH (
  format = 'ORC'  -- ORC, PARQUET, AVRO, JSON, TEXTFILE, etc.
);
```

**Compression:**
- Default: GZIP
- Configure: `hive.compression-codec=SNAPPY` in catalog properties

### Non-Relational Data Sources

**JMX Connector:**
```properties
connector.name=jmx
```

```sql
SHOW TABLES FROM jmx.current;
DESCRIBE jmx.current."java.lang:type=runtime";
SELECT vname, uptime, node FROM jmx.current."java.lang:type=runtime";
```

**Black Hole Connector (for testing):**
```properties
connector.name=blackhole
```

```sql
CREATE SCHEMA blackhole.test;
CREATE TABLE blackhole.test.orders AS SELECT * from tpch.tiny.orders;
INSERT INTO blackhole.test.orders SELECT * FROM tpch.sf3.orders;
```

**Memory Connector:**
```properties
connector.name=memory
```

**Note:** Data stored in memory only; lost on cluster stop.

---
