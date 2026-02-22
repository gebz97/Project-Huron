
## Chapter 8: Using SQL in Trino

### Trino Statements

| Statement | Description |
|-----------|-------------|
| `SHOW CATALOGS [LIKE pattern]` | List catalogs |
| `SHOW SCHEMAS [FROM catalog] [LIKE pattern]` | List schemas |
| `SHOW TABLES [FROM schema] [LIKE pattern]` | List tables |
| `SHOW FUNCTIONS` | List SQL functions |
| `SHOW COLUMNS FROM table` or `DESCRIBE table` | Show table columns |
| `USE catalog.schema` | Set default catalog/schema |
| `SHOW STATS FOR table` | Show table statistics |
| `EXPLAIN [ (option) ] query` | Show query plan |

**EXPLAIN Options:**
- `FORMAT { TEXT | GRAPHVIZ | JSON }`
- `TYPE { LOGICAL | DISTRIBUTED | IO | VALIDATE }`

### System Tables

**Runtime Queries:**
```sql
DESCRIBE system.runtime.queries;
SELECT * FROM system.runtime.queries WHERE state='FAILED';
SELECT * FROM system.runtime.queries WHERE state='RUNNING';
```

**Kill Query:**
```sql
CALL system.runtime.kill_query(query_id => 'queryId', message => 'Killed');
```

**Cluster Nodes:**
```sql
SELECT * FROM system.runtime.nodes;
```

### Catalogs, Schemas, Tables

**Catalog Properties:**
```sql
SELECT * FROM system.metadata.schema_properties;
SELECT * FROM system.metadata.table_properties;
SELECT * FROM system.metadata.column_properties;
```

**Create Schema:**
```sql
CREATE SCHEMA [IF NOT EXISTS] schema_name
[WITH (property_name = expression [, ...])];

-- Example with Hive
CREATE SCHEMA hive.web WITH (location = 's3://example-org/web/');
```

**Alter Schema:**
```sql
ALTER SCHEMA name RENAME TO new_name;
DROP SCHEMA [IF EXISTS] schema_name;
```

### Information Schema

```sql
SHOW TABLES IN system.information_schema;

SELECT * FROM hive.information_schema.tables;
SELECT table_catalog, table_schema, table_name, column_name
FROM hive.information_schema.columns
WHERE table_name = 'nation';
```

### Tables

**Create Table:**
```sql
CREATE TABLE [IF NOT EXISTS] table_name (
  column_name data_type [COMMENT comment] [WITH (property = value)],
  ...
) [COMMENT table_comment]
[WITH (property = expression [, ...])];
```

**Table Properties (Hive connector):**

| Property | Description |
|----------|-------------|
| `external_location` | Filesystem location for external table |
| `format` | File format (ORC, PARQUET, AVRO, etc.) |
| `partitioned_by` | Array of partition columns |

**Copy Table (LIKE):**
```sql
CREATE TABLE hive.web.page_view_bucketed (
  comment VARCHAR,
  LIKE hive.web.page_views INCLUDING PROPERTIES
) WITH (
  bucketed_by = ARRAY['user_id'],
  bucket_count = 50
);
```

**CTAS (Create Table As Select):**
```sql
CREATE TABLE hive.web.page_views_orc_part
WITH (
  format = 'ORC',
  partitioned_by = ARRAY['view_date','country']
)
AS SELECT * FROM hive.web.page_view_text;

-- Create empty copy
CREATE TABLE hive.web.user_session
AS SELECT * FROM page_views
WITH NO DATA;
```

**ALTER TABLE:**
```sql
ALTER TABLE name RENAME TO new_name;
ALTER TABLE name ADD COLUMN column_name data_type;
ALTER TABLE name DROP COLUMN column_name;
ALTER TABLE name RENAME COLUMN column_name TO new_column_name;
```

**DROP TABLE:**
```sql
DROP TABLE [IF EXISTS] table_name;
```

### Data Types

| Category | Types |
|----------|-------|
| **Boolean** | `BOOLEAN` |
| **Integer** | `TINYINT`, `SMALLINT`, `INTEGER`/`INT`, `BIGINT` |
| **Floating-point** | `REAL`, `DOUBLE` |
| **Fixed-precision** | `DECIMAL` |
| **String** | `VARCHAR`/`VARCHAR(n)`, `CHAR`/`CHAR(n)` |
| **Collection** | `ARRAY`, `MAP`, `JSON`, `ROW` |
| **Temporal** | `DATE`, `TIME`, `TIME WITH TIMEZONE`, `TIMESTAMP`, `TIMESTAMP WITH TIMEZONE`, `INTERVAL YEAR TO MONTH`, `INTERVAL DAY TO SECOND` |

**String Behavior:**
```sql
-- VARCHAR vs CHAR
SELECT length(cast('hello' AS char(10)));  -- 10 (padded)
SELECT length(cast('hello' AS varchar(10))); -- 5 (exact)

-- Insertion
CREATE TABLE varchars(col varchar(5));
INSERT INTO varchars values('12345');  -- OK
INSERT INTO varchars values('123456'); -- Error
```

### Temporal Data Types

**Parsing Formats:**

| Type | Accepted Formats |
|------|------------------|
| `TIMESTAMP` | yyyy-M-d, yyyy-M-d H:m, yyyy-M-d H:m:s, yyyy-M-d H:m:s.SSS |
| `TIME` | H:m, H:m:s, H:m:s.SSS |
| `DATE` | YYYY-MM-DD |

**Time Zone Examples:**
```sql
SELECT TIME '02:56:15 UTC' AT TIME ZONE 'America/Los_Angeles';
-- 18:56:15.000 America/Los_Angeles

SELECT TIMESTAMP '1983-10-19 07:30:05.123 America/New_York' AT TIME ZONE 'UTC';
-- 1983-10-19 11:30:05.123 UTC
```

**Intervals:**
```sql
SELECT INTERVAL '1-2' YEAR TO MONTH;  -- 1-2
SELECT INTERVAL '4' MONTH;             -- 0-4
SELECT INTERVAL '4 01:03:05.44' DAY TO SECOND; -- 4 01:03:05.440
```

### Type Casting

```sql
-- CAST
SELECT CAST('2019-01-01' AS DATE);
SELECT CAST('1' AS INTEGER);

-- try_cast (returns NULL on error)
SELECT try_cast('a' AS INTEGER);  -- NULL
```

### SELECT Statement

```sql
[WITH with_query [, ...]]
SELECT [ALL | DISTINCT] select_expr [, ...]
[FROM from_item [, ...]]
[WHERE condition]
[GROUP BY [ALL | DISTINCT] grouping_element [, ...]]
[HAVING condition]
[{UNION | INTERSECT | EXCEPT} [ALL | DISTINCT] select]
[ORDER BY expression [ASC | DESC] [, ...]]
[LIMIT [count | ALL]]
```

**WHERE Clause:**
```sql
SELECT custkey, acctbal FROM tpch.sf1.customer
WHERE acctbal < 0;

SELECT custkey, acctbal FROM tpch.sf1.customer
WHERE acctbal > 0 AND acctbal < 500;
```

**GROUP BY and HAVING:**
```sql
SELECT mktsegment, round(sum(acctbal) / 1000000, 3) AS acctbal_millions
FROM tpch.sf1.customer
GROUP BY mktsegment
HAVING sum(acctbal) > 1300000;
```

**ORDER BY and LIMIT:**
```sql
SELECT mktsegment, round(sum(acctbal), 2) AS acctbal_per_mktsegment
FROM tpch.sf1.customer
GROUP BY mktsegment
HAVING sum(acctbal) > 0
ORDER BY acctbal_per_mktsegment DESC
LIMIT 1;
```

**JOIN:**
```sql
-- Explicit JOIN
SELECT custkey, mktsegment, nation.name AS nation
FROM tpch.tiny.nation
JOIN tpch.tiny.customer ON nation.nationkey = customer.nationkey;

-- Implicit CROSS JOIN with WHERE
SELECT custkey, mktsegment, nation.name AS nation
FROM tpch.tiny.nation, tpch.tiny.customer
WHERE nation.nationkey = customer.nationkey;
```

**UNION, INTERSECT, EXCEPT:**
```sql
SELECT * FROM (VALUES 1, 2)
UNION
SELECT * FROM (VALUES 2, 3);
-- 1, 2, 3

SELECT * FROM (VALUES 1, 2)
UNION ALL
SELECT * FROM (VALUES 2, 3);
-- 1, 2, 2, 3

SELECT * FROM (VALUES 1, 2)
INTERSECT
SELECT * FROM (VALUES 2, 3);
-- 2

SELECT * FROM (VALUES 1, 2)
EXCEPT
SELECT * FROM (VALUES 2, 3);
-- 1
```

**GROUPING SETS, ROLLUP, CUBE:**
```sql
SELECT mktsegment,
       round(sum(acctbal), 2) AS total_acctbal,
       GROUPING(mktsegment) AS id
FROM tpch.tiny.customer
GROUP BY ROLLUP (mktsegment)
ORDER BY id, total_acctbal;
```

**WITH Clause (CTE):**
```sql
WITH
total AS (
  SELECT mktsegment, round(sum(acctbal)) AS total_per_mktsegment
  FROM tpch.tiny.customer
  GROUP BY 1
),
average AS (
  SELECT round(avg(total_per_mktsegment)) AS average
  FROM total
)
SELECT mktsegment, total_per_mktsegment, average
FROM total, average
WHERE total_per_mktsegment > average;
```

**Subqueries:**

```sql
-- Scalar subquery
SELECT regionkey, name
FROM tpch.tiny.nation
WHERE regionkey = (
  SELECT regionkey FROM tpch.tiny.region WHERE name = 'AMERICA'
);

-- EXISTS
SELECT name FROM nation n
WHERE EXISTS (SELECT 1 FROM region r WHERE r.regionkey = n.regionkey);

-- IN / ANY
SELECT name FROM nation
WHERE regionkey IN (SELECT regionkey FROM region);

-- ALL
SELECT name FROM nation
WHERE regionkey <> ALL (SELECT regionkey FROM region WHERE regionkey > 2);
```

### DELETE

```sql
DELETE FROM table_name [WHERE condition];
DELETE FROM hive.web.page_views WHERE view_date = DATE '2019-01-14';
```

---
