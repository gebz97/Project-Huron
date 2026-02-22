
## Chapter 7: Advanced Connector Examples

### HBase with Phoenix

**Catalog file (etc/catalog/bigtables.properties):**
```properties
connector.name=phoenix
phoenix.connection-url=jdbc:phoenix:zookeeper1,zookeeper2:2181:/hbase
```

**Usage:**
```sql
SHOW SCHEMAS FROM bigtable;
SHOW TABLES FROM bigtable.example;
SHOW COLUMNS FROM bigtable.example.user;
```

### Accumulo (Key-Value Store)

**Accumulo Data Model:**
```
Key = (row ID, column, timestamp) → Value
```

**Catalog file (etc/catalog/accumulo.properties):**
```properties
connector.name=accumulo
accumulo.instance=accumulo
accumulo.zookeepers=zookeeper.example.com:2181
accumulo.username=user
accumulo.password=password
```

**Create Table:**
```sql
CREATE TABLE accumulo.ontime.flights (
  rowid VARCHAR,
  flightdate VARCHAR,
  flightnum INTEGER,
  origin VARCHAR,
  dest VARCHAR
);

-- With column families
CREATE TABLE accumulo.ontime.flights (
  rowid VARCHAR,
  flightdate VARCHAR,
  flightnum INTEGER,
  origin VARCHAR,
  dest VARCHAR
) WITH (
  column_mapping = 'origin:location:origin,dest:location:dest'
);
```

**External Table:**
```sql
CREATE TABLE accumulo.ontime.flights (
  rowid VARCHAR,
  flightdate VARCHAR,
  flightnum INTEGER,
  origin VARCHAR,
  dest VARCHAR
) WITH (
  external = true,
  column_mapping = 'origin:location:origin,dest:location:dest'
);
```

**Enable Indexing:**
```sql
CREATE TABLE accumulo.ontime.flights (
  rowid VARCHAR,
  flightdate VARCHAR,
  flightnum INTEGER,
  origin VARCHAR,
  dest VARCHAR
) WITH (
  index_columns = 'flightdate,origin'
);
```

### Cassandra Connector

**Catalog file (etc/catalog/sitedata.properties):**
```properties
connector.name=cassandra
cassandra.contact-points=sitedata.example.com
```

**Usage:**
```sql
SELECT * FROM sitedata.cart.users;
```

### Kafka Connector

**Catalog file (etc/catalog/trafficstream.properties):**
```properties
connector.name=kafka
kafka.table-names=web.pages,web.users
kafka.nodes=trafficstream.example.com:9092
```

**Usage:**
```sql
-- Query live Kafka topics
SELECT * FROM trafficstream.web.pages;
SELECT * FROM trafficstream.web.users;

-- Migrate to HDFS
CREATE TABLE hdfs.web.pages WITH (
  format = 'ORC',
  partitioned_by = ARRAY['view_date']
) AS SELECT * FROM trafficstream.web.pages;

-- Incremental insert
INSERT INTO hdfs.web.pages
SELECT * FROM trafficstream.web.pages
WHERE _partition_offset > last_offset;
```

### Elasticsearch Connector

**Catalog file (etc/catalog/search.properties):**
```properties
connector.name=elasticsearch
elasticsearch.host=searchcluster.example.com
```

**Usage:**
```sql
DESCRIBE search.default.server;
SELECT * FROM "blogs: +trino";  -- Full-text search
```

### Federated Queries

**Example: Join Hive (S3) and PostgreSQL**

```sql
-- Top carriers with names (not just codes)
SELECT f.uniquecarrier, c.description, count(*) AS ct
FROM hive.ontime.flights_orc f, postgresql.airline.carrier c
WHERE c.code = f.uniquecarrier
GROUP BY f.uniquecarrier, c.description
ORDER BY count(*) DESC
LIMIT 10;

-- Airports with names and cities
SELECT f.origin, c.name, c.city, count(*) AS ct
FROM hive.ontime.flights_orc f, postgresql.airline.airport c
WHERE c.code = f.origin AND c.state = 'AK'
GROUP BY origin, c.name, c.city
ORDER BY count(*) DESC
LIMIT 10;
```

---
