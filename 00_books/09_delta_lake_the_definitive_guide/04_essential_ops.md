
## Chapter 3: Essential Delta Lake Operations

### Create Operations

**Create Table with DeltaTable API:**
```python
from delta.tables import DeltaTable

(DeltaTable.createIfNotExists(spark)
 .tableName("exampleDB.countries")
 .addColumn("id", "INT")
 .addColumn("country", "STRING")
 .addColumn("capital", "STRING")
 .execute())
```

**Create Table with Spark SQL:**
```sql
CREATE TABLE exampleDB.countries (
  id INT,
  country STRING,
  capital STRING
) USING DELTA;
```

### Loading Data

**INSERT INTO:**
```sql
INSERT INTO exampleDB.countries VALUES 
  (1, 'United Kingdom', 'London'),
  (2, 'Canada', 'Toronto');
```

**PySpark insertInto:**
```python
data = [(1, "United Kingdom", "London"), (2, "Canada", "Toronto")]
schema = ["id", "country", "capital"]
df = spark.createDataFrame(data, schema=schema)

df.write.format("delta").insertInto("exampleDB.countries")
```

**Append Mode:**
```python
data = [(3, 'United States', 'Washington, D.C.')]
df = spark.createDataFrame(data, schema=schema)

(df.write
 .format("delta")
 .mode("append")
 .saveAsTable("exampleDB.countries"))
```

**CREATE TABLE AS SELECT (CTAS):**
```sql
CREATE TABLE exampleDB.countries2 AS 
SELECT * FROM exampleDB.countries;
```

### Transaction Log

**Structure:**
```
countries.delta/
└── _delta_log/
    └── 00000000000000000000.json
```

**Contents:**
- Creation info
- Processing engine used
- Number of records
- Write metrics
- Deletion information
- Maintenance operations

### Read Operations

**Basic SELECT:**
```sql
SELECT * FROM exampleDB.countries;
```

**DeltaTable API:**
```python
from delta.tables import DeltaTable
delta_table = DeltaTable.forName(spark, "exampleDB.countries")
delta_table.toDF().show()
```

**Filtering:**
```sql
SELECT * FROM exampleDB.countries WHERE capital = 'London';
```

```python
delta_table_df.filter(delta_table_df.capital == 'London')
```

**Select Columns:**
```sql
SELECT id, capital FROM exampleDB.countries;
```

```python
delta_table_df.select("id", "capital")
```

### Time Travel

**By Version:**
```sql
SELECT DISTINCT id FROM exampleDB.countries VERSION AS OF 1;
```

```python
(spark.read
 .option("versionAsOf", "1")
 .load("countries.delta")
 .select("id")
 .distinct()
 .show())
```

**By Timestamp:**
```sql
SELECT count(1) FROM exampleDB.countries TIMESTAMP AS OF "2024-04-20";
```

```python
(spark.read
 .option("timestampAsOf", "2024-04-20")
 .load("countries.delta")
 .count())
```

### Update Operations

**SQL UPDATE:**
```sql
UPDATE exampleDB.countries 
SET country = 'U.K.' 
WHERE id = 1;
```

**Python DeltaTable.update():**
```python
delta_table.update(
  condition = "id = 1",
  set = {"country": "U.K."}
)
```

### Delete Operations

**SQL DELETE:**
```sql
DELETE FROM exampleDB.countries WHERE id = 1;
```

**Python DeltaTable.delete():**
```python
from pyspark.sql.functions import col

delta_table.delete("id = 1")           # SQL expression
delta_table.delete(col("id") == 2)      # PySpark expression
```

**⚠️ Warning:** No WHERE clause = delete ALL records

### Overwrite Operations

**Overwrite Mode:**
```python
data = [(1, 'India', 'New Delhi'), (4, 'Australia', 'Canberra')]
df = spark.createDataFrame(data, schema=schema)

(df.write
 .format("delta")
 .mode("overwrite")
 .saveAsTable("exampleDB.countries"))
```

**INSERT OVERWRITE:**
```sql
INSERT OVERWRITE exampleDB.countries 
VALUES (3, 'U.S.', 'Washington, D.C.');
```

### Merge (Upsert) Operations

**SQL MERGE:**
```sql
MERGE INTO exampleDB.countries A
USING (SELECT * FROM parquet.`countries.parquet`) B
ON A.id = B.id
WHEN MATCHED THEN 
  UPDATE SET id = A.id, country = B.country, capital = B.capital
WHEN NOT MATCHED THEN 
  INSERT (id, country, capital) VALUES (B.id, B.country, B.capital);
```

**Python DeltaMergeBuilder:**
```python
idf = spark.createDataFrame(
  [(1, 'India', 'New Delhi'), (4, 'Australia', 'Canberra')],
  schema=schema
)

(delta_table.alias("target")
 .merge(
   source=idf.alias("source"),
   condition="source.id = target.id"
 )
 .whenMatchedUpdate(set={
   "country": "source.country",
   "capital": "source.capital"
 })
 .whenNotMatchedInsert(values={
   "id": "source.id",
   "country": "source.country",
   "capital": "source.capital"
 })
 .execute())
```

### Parquet Conversions

**Regular Parquet:**
```sql
CONVERT TO DELTA parquet.`countries.parquet`;
```

```python
from delta.tables import DeltaTable
delta_table = DeltaTable.convertToDelta(
  spark, 
  "parquet.`countries.parquet`"
)
```

**Iceberg to Delta:**
```sql
CONVERT TO DELTA iceberg.`countries.iceberg`;
```

### Metadata and History

**DESCRIBE DETAIL:**
```sql
DESCRIBE DETAIL exampleDB.countries;
```

```python
delta_table_detail = delta_table.detail()
```

**Returns:**
- Schema
- Reader/writer version
- Table properties
- Last modified time
- Number of files

**DESCRIBE HISTORY:**
```sql
DESCRIBE HISTORY exampleDB.countries;
```

```python
delta_table_history = delta_table.history()
```

**Transaction-level metadata:**
- Operation type
- User
- Timestamp
- Metrics

---
