
## Chapter 9: Apache Flink

### Configuration Setup

```bash
# Download Flink
FLINK_VERSION=1.16.1
SCALA_VERSION=2.12
wget https://archive.apache.org/dist/flink/flink-${FLINK_VERSION}/flink-${FLINK_VERSION}-bin-scala_${SCALA_VERSION}.tgz
tar xzvf flink-${FLINK_VERSION}-bin-scala_${SCALA_VERSION}.tgz

# Download Iceberg Flink runtime
# Place in FLINK_HOME/lib directory

# Download Hadoop (if needed)
HADOOP_VERSION=2.8.5
wget https://archive.apache.org/dist/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz
tar xzvf hadoop-${HADOOP_VERSION}.tar.gz

# Set environment variables
export HADOOP_HOME=`pwd`/hadoop-${HADOOP_VERSION}
export HADOOP_CLASSPATH=`$HADOOP_HOME/bin/hadoop classpath`

# Start Flink cluster
./bin/start-cluster.sh

# Start Flink SQL Client
./bin/sql-client.sh embedded
```

### Flink SQL DDL

**CREATE CATALOG - Hadoop:**
```sql
CREATE CATALOG local_catalog WITH (
  'type'='iceberg',
  'catalog-type'='hadoop',
  'warehouse'='hdfs://nn:8020/warehouse/path'
);

USE CATALOG local_catalog;
```

**CREATE CATALOG - Hive:**
```sql
CREATE CATALOG hive_catalog WITH (
  'type'='iceberg',
  'catalog-type'='hive',
  'uri'='thrift://localhost:9083',
  'warehouse'='hdfs://nn:8020/warehouse/path'
);

USE CATALOG hive_catalog;
```

**CREATE CATALOG - Custom:**
```sql
CREATE CATALOG custom_catalog WITH (
  'type'='iceberg',
  'catalog-impl'='com.my.custom.CatalogImpl',
  'my-additional-catalog-config'='my-value'
);
```

**CREATE DATABASE:**
```sql
CREATE DATABASE iceberg_db;
USE iceberg_db;
```

**CREATE TABLE:**
```sql
CREATE TABLE employee (
  id BIGINT,
  role STRING,
  department STRING,
  salary FLOAT,
  region STRING
) WITH (
  'connector'='iceberg'
);
```

**CREATE TABLE PARTITIONED BY:**
```sql
CREATE TABLE emp_partitioned (
  id BIGINT,
  role STRING
) PARTITIONED BY (role) WITH (
  'connector'='iceberg'
);
```

**CREATE TABLE LIKE:**
```sql
CREATE TABLE emp_like LIKE employee;
```

**ALTER TABLE - Rename:**
```sql
ALTER TABLE employee RENAME TO emp_new;
```

**ALTER TABLE - Set Properties:**
```sql
ALTER TABLE employee SET ('write.format.default'='avro');
```

**DROP TABLE:**
```sql
DROP TABLE employee;
```

### Reading Data (Flink SQL)

**Batch Read:**
```sql
SET execution.runtime-mode = batch;
SELECT * FROM employee;
```

**Streaming Read:**
```sql
SET execution.runtime-mode = streaming;
SELECT * FROM employee /*+ OPTIONS('streaming'='true', 'monitor-interval'='1s')*/;
```

**Metadata Tables:**
```sql
-- History
SELECT * FROM `catalog`.`database`.`table`$history;

-- Metadata logs
SELECT * FROM `catalog`.`database`.`table`$metadata_log_entries;

-- Snapshots
SELECT * FROM `catalog`.`database`.`table`$snapshots;
```

### Writing Data (Flink SQL)

**INSERT INTO:**
```sql
INSERT INTO employee VALUES (1, 'Software Engineer', 'Engineering', 25000, 'NA');
INSERT INTO employee SELECT id, role FROM emp_new;
```

**INSERT OVERWRITE:**
```sql
-- Overwrite entire table
INSERT OVERWRITE employee VALUES (1, 'Software Tester', 'Engineering', 23000, 'NA');

-- Overwrite partition
INSERT OVERWRITE employee PARTITION(department='Engineering')
SELECT * FROM updated_emp_data WHERE department='Engineering';
```

**UPSERT (enable at table creation):**
```sql
CREATE TABLE employee (
  id INT UNIQUE,
  role STRING NOT NULL,
  department STRING NOT NULL,
  salary FLOAT,
  region STRING NOT NULL,
  PRIMARY KEY(id) NOT ENFORCED
) WITH ('format-version'='2', 'write.upsert.enabled'='true');

INSERT INTO employee VALUES (1, 'Director', 'Product', 33000, 'APAC');
```

**UPSERT (per query):**
```sql
INSERT INTO employee /*+ OPTIONS('upsert-enabled'='true') */ 
VALUES (3, 'Manager', 'Engineering', 26000, 'NA');
```

### Flink Java API

**Maven dependencies (pom.xml):**
```xml
<dependencies>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-java</artifactId>
    <version>1.16.1</version>
  </dependency>
  <dependency>
    <groupId>org.apache.iceberg</groupId>
    <artifactId>iceberg-flink-runtime-1.16</artifactId>
    <version>1.3.0</version>
  </dependency>
</dependencies>
```

**Java example:**
```java
public class App {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
    
    // Create catalog
    tableEnv.executeSql(
      "CREATE CATALOG iceberg WITH (" +
      "'type'='iceberg'," +
      "'catalog-impl'='org.apache.iceberg.nessie.NessieCatalog'," +
      "'uri'='http://catalog:19120/api/v1'," +
      "'warehouse'='s3://warehouse'" +
      ")"
    );
    
    tableEnv.useCatalog("iceberg");
    
    // Create database and table
    tableEnv.executeSql("CREATE DATABASE IF NOT EXISTS db");
    tableEnv.executeSql(
      "CREATE TABLE IF NOT EXISTS db.employees (" +
      "id BIGINT, department STRING, salary BIGINT)"
    );
    
    // Create DataStream
    DataStream<Row> dataStream = env.fromElements(
      Row.of(1L, "Engineering", 100000L),
      Row.of(2L, "Sales", 80000L)
    );
    
    // Convert to Table and insert
    Table inputTable = tableEnv.fromDataStream(dataStream);
    inputTable.executeInsert("db.employees");
    
    env.execute();
  }
}
```

---
