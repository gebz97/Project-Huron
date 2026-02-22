
## Chapter 6: Apache Spark

### Configuration

**Spark Shell:**
```bash
spark-shell --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0
```

**Spark SQL:**
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0
```

**Add JAR directly:**
```bash
./bin/spark-shell --jars /path/to/iceberg-spark-runtime.jar
```

**PySpark:**
```python
from pyspark.sql import SparkSession
from pyspark import SparkConf

conf = SparkConf()
conf.set("spark.jars.packages", "org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0")

spark = SparkSession.builder.config(conf=conf).getOrCreate()
```

### Catalog Configuration

**SparkCatalog Types:**

| Type | Description |
|------|-------------|
| `hive` | Uses Hive Metastore |
| `hadoop` | Directory-based (filesystem) |
| `custom` | Custom implementation |

**Hive Catalog Example:**
```python
spark.sql("""
CREATE CATALOG hive_catalog WITH (
  'type'='iceberg',
  'catalog-type'='hive',
  'uri'='thrift://metastore-host:9083'
)
""")
```

**AWS Glue Example:**
```python
spark.conf.set("spark.sql.catalog.glue", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.glue.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog")
spark.conf.set("spark.sql.catalog.glue.warehouse", "s3://my-bucket/warehouse/")
spark.conf.set("spark.sql.catalog.glue.io-impl", "org.apache.iceberg.aws.s3.S3FileIO")
```

### DDL Operations

**CREATE TABLE:**
```sql
CREATE TABLE glue.test.employee (
  id INT,
  role STRING,
  department STRING,
  salary FLOAT,
  region STRING
) USING iceberg;
```

**CREATE TABLE with partitions:**
```sql
CREATE TABLE glue.test.emp_partitioned (
  id INT,
  role STRING,
  department STRING
) USING iceberg
PARTITIONED BY (department);
```

**Hidden Partitioning:**
```sql
CREATE TABLE glue.test.emp_partitioned_month (
  id INT,
  role STRING,
  department STRING,
  join_date DATE
) USING iceberg
PARTITIONED BY (months(join_date));
```

**CREATE TABLE AS SELECT (CTAS):**
```sql
CREATE TABLE glue.test.employee_ctas
USING iceberg
AS SELECT * FROM glue.test.sample;
```

**ALTER TABLE - Rename:**
```sql
ALTER TABLE glue.test.employee RENAME TO glue.test.emp_renamed;
```

**ALTER TABLE - Set Properties:**
```sql
ALTER TABLE glue.test.employee 
SET TBLPROPERTIES ('write.wap.enabled'='true');
```

**ALTER TABLE - Add Column:**
```sql
ALTER TABLE glue.test.employee ADD COLUMN manager STRING;

-- Add multiple
ALTER TABLE glue.test.employee ADD COLUMN details STRING, manager_id INT;

-- Add at specific position
ALTER TABLE glue.test.employee ADD COLUMN new_column BIGINT AFTER department;
ALTER TABLE glue.test.employee ADD COLUMN first_column BIGINT FIRST;
```

**ALTER TABLE - Rename Column:**
```sql
ALTER TABLE glue.test.employee RENAME COLUMN role TO title;
```

**ALTER TABLE - Modify Column:**
```sql
ALTER TABLE glue.test.employee ALTER COLUMN id TYPE BIGINT;

-- Reorder
ALTER TABLE glue.test.employee ALTER COLUMN salary FIRST;
```

**ALTER TABLE - Drop Column:**
```sql
ALTER TABLE glue.test.employee DROP COLUMN department;
```

**Spark SQL Extensions - Add Partition Field:**
```sql
ALTER TABLE glue.test.employee ADD PARTITION FIELD region;
```

**Spark SQL Extensions - Drop Partition Field:**
```sql
ALTER TABLE glue.test.employee DROP PARTITION FIELD department;
```

**Spark SQL Extensions - Set Write Order:**
```sql
ALTER TABLE glue.test.employee WRITE ORDERED BY id ASC;
```

**Spark SQL Extensions - Set Write Distribution:**
```sql
ALTER TABLE glue.test.employee WRITE DISTRIBUTED BY PARTITION;
```

**Spark SQL Extensions - Set Identifier Fields:**
```sql
ALTER TABLE glue.test.employee SET IDENTIFIER FIELDS id;
ALTER TABLE glue.test.employee DROP IDENTIFIER FIELDS id;
```

**DROP TABLE:**
```sql
-- Remove from catalog only (keep data)
DROP TABLE glue.test.employee;

-- Remove from catalog and delete data
DROP TABLE glue.test.employee PURGE;
```

### Reading Data

**Select All:**
```sql
SELECT * FROM glue.test.employee;
```

```python
df = spark.table("glue.test.employee")
```

**Filter Rows:**
```sql
SELECT * FROM glue.test.employee WHERE department = 'Marketing';
```

```python
filtered_df = df.filter(df['department'] == 'Marketing')
```

**Count Records:**
```sql
SELECT COUNT(*) FROM glue.test.employee;
```

```python
print(df.count())
```

**Average:**
```sql
SELECT AVG(salary) FROM glue.test.employee;
```

```python
df.agg({'salary': 'avg'}).show()
```

**Sum:**
```sql
SELECT SUM(salary) FROM glue.test.employee;
```

```python
df.agg({'salary': 'sum'}).show()
```

**Maximum by Group:**
```sql
SELECT category, MAX(salary) FROM glue.test.employee GROUP BY category;
```

```python
df.groupBy("category").max("salary").show()
```

**Window Functions:**
```sql
SELECT *, RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM glue.test.employee;
```

```python
from pyspark.sql import Window
from pyspark.sql.functions import row_number

windowSpec = Window.partitionBy(df['department']).orderBy(df['salary'].desc())
df.withColumn("rank", row_number().over(windowSpec)).show()
```

### Writing Data

**INSERT INTO:**
```sql
INSERT INTO glue.test.employee VALUES 
  (1, 'Software Engineer', 'Engineering', 25000, 'NA'),
  (2, 'Director', 'Sales', 22000, 'EMEA');
```

```python
from pyspark.sql import Row

data = [
  Row(id=1, role='Software Engineer', department='Engineering', salary=25000, region='NA'),
  Row(id=2, role='Director', department='Sales', salary=22000, region='EMEA')
]
df = spark.createDataFrame(data)
df.writeTo("glue.test.employee").append()
```

**MERGE INTO:**
```sql
MERGE INTO glue.test.employee AS target
USING employee_updates AS source ON target.id = source.id
WHEN MATCHED AND source.role = 'Manager' AND source.salary > 100000 THEN
  UPDATE SET target.salary = source.salary
WHEN NOT MATCHED THEN
  INSERT *;
```

**INSERT OVERWRITE - Static:**
```sql
INSERT OVERWRITE glue.test.employees
PARTITION (region = 'EMEA')
SELECT * FROM employee_source WHERE region = 'EMEA';
```

**INSERT OVERWRITE - Dynamic:**
```sql
SET spark.sql.sources.partitionOverwriteMode = dynamic;

INSERT OVERWRITE glue.test.employee
SELECT * FROM employee_source WHERE region = 'EMEA';
```

**DELETE FROM:**
```sql
DELETE FROM glue.test.employee WHERE id < 3;
```

**UPDATE:**
```sql
UPDATE glue.test.employee
SET region = 'APAC', salary = 6000
WHERE id = 6;

-- With subquery
UPDATE glue.test.employee AS e
SET region = 'NA'
WHERE EXISTS (SELECT id FROM emp_history WHERE emp_history.id = e.id);
```

### Table Maintenance Procedures

**Expire Snapshots:**
```sql
CALL glue.system.expire_snapshots('test.employees', date_sub(current_date(), 90), 50);
```

**Rewrite Datafiles:**
```sql
CALL glue.system.rewrite_data_files('test.employee');
```

**Rewrite Manifests:**
```sql
CALL test.system.rewrite_manifests('test.employee');
```

**Remove Orphan Files:**
```sql
CALL glue.system.remove_orphan_files(table => 'test.employee', dry_run => true);
```

---
