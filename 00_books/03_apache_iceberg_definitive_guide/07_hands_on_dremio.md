
## Chapter 7: Dremio's SQL Query Engine

### Configuration

**Dremio Source to Iceberg Catalog Mapping:**

| Dremio Source | Iceberg Catalog |
|---------------|-----------------|
| AWS Glue | AWS Glue |
| Amazon S3, ADLS, HDFS, GCS | Hadoop |
| Arctic/Nessie | Nessie |

### DDL Operations

**CREATE TABLE:**
```sql
CREATE TABLE employee (id INT, role VARCHAR, department VARCHAR, salary FLOAT, region VARCHAR);
```

**CREATE TABLE AS SELECT:**
```sql
CREATE TABLE employee AS SELECT * FROM mySource."myFolder"."empData.csv";
```

**CREATE TABLE with partitioning and sorting:**
```sql
CREATE TABLE emp_partitioned (id INT, role VARCHAR, department VARCHAR)
PARTITION BY (department)
LOCALSORT BY (id);
```

**CREATE TABLE with partition transforms:**
```sql
CREATE TABLE emp_partitioned_by_month
(id INT, role VARCHAR, department VARCHAR, join_date DATE)
PARTITION BY (month(join_date));
```

**CREATE TABLE with row access policy:**
```sql
CREATE FUNCTION restrict_region(region VARCHAR)
RETURNS BOOLEAN
RETURN SELECT CASE 
  WHEN query_user()='user@example.com' OR is_member('HR') THEN true
  WHEN region = 'North' THEN true
  ELSE false
END;

CREATE TABLE regional_employee_data (
  id INT,
  role VARCHAR,
  department VARCHAR,
  salary FLOAT,
  region VARCHAR,
  ROW ACCESS POLICY restrict_region(region)
);
```

**CREATE TABLE with column masking:**
```sql
CREATE FUNCTION mask_salary(salary VARCHAR)
RETURNS VARCHAR
RETURN SELECT CASE 
  WHEN query_user()='user@example.com' OR is_member('HR') THEN salary
  ELSE 'XXX-XX'
END;

CREATE TABLE employee_salaries (
  id INT,
  salary VARCHAR MASKING POLICY mask_salary(salary),
  department VARCHAR
);
```

**ALTER TABLE - Add Columns:**
```sql
ALTER TABLE employee ADD COLUMNS (date_of_birth DATE);
```

**ALTER TABLE - Modify Column (set masking):**
```sql
ALTER TABLE employee MODIFY COLUMN ssn_col SET MASKING POLICY protect_ssn;
ALTER TABLE employee MODIFY COLUMN ssn_col UNSET MASKING POLICY;
```

**ALTER TABLE - Rename Column:**
```sql
ALTER TABLE employee ALTER COLUMN role title VARCHAR;
```

**ALTER TABLE - Drop Column:**
```sql
ALTER TABLE employee DROP COLUMN department;
```

**DROP TABLE:**
```sql
DROP TABLE employee;
```

### Reading Data

**SELECT:**
```sql
SELECT * FROM employee;
```

**Filter:**
```sql
SELECT * FROM employee WHERE department = 'Engineering';
```

**Count Records:**
```sql
SELECT role, COUNT(*) as employee_count FROM employee GROUP BY role;
```

**Average:**
```sql
SELECT region, AVG(salary) FROM employee GROUP BY region;
```

**Sum:**
```sql
SELECT department, SUM(salary) as total_salary FROM employee GROUP BY department;
```

**Maximum:**
```sql
SELECT department, MAX(salary) as highest_salary FROM employee GROUP BY department;
```

**Window Functions:**
```sql
SELECT salary, region,
RANK() OVER (PARTITION BY region ORDER BY salary DESC) AS salary_rank
FROM employee;
```

### Writing Data

**INSERT INTO:**
```sql
INSERT INTO employee VALUES
  (7, 'Solution Architect', 'Sales', 15000, 'EMEA'),
  (8, 'Product Manager', 'Product', 28000, 'NA');
```

**COPY INTO:**
```sql
COPY INTO employee FROM '@mySource/myFolder/' FILE_FORMAT 'csv';
COPY INTO employee FROM '@mySource/myFolder/employee_data.csv';
```

**MERGE INTO:**
```sql
MERGE INTO employee AS e USING new_employee AS ne ON (e.id = ne.id)
WHEN MATCHED THEN UPDATE SET salary = ne.salary
WHEN NOT MATCHED THEN INSERT (id, role, department, salary, region) 
  VALUES (ne.id, ne.role, ne.department, ne.salary, ne.region);
```

**DELETE:**
```sql
DELETE FROM employee WHERE region = 'NA';
```

**UPDATE:**
```sql
UPDATE employee SET salary = salary + 2000 WHERE department = 'Marketing';
```

### Table Maintenance

**Expire Snapshots:**
```sql
VACUUM TABLE 'employee' 
EXPIRE SNAPSHOTS older_than TIMESTAMP '2023-07-10 00:00:00.000' retain_last 30;
```

**Rewrite Datafiles:**
```sql
OPTIMIZE TABLE 'employee' REWRITE DATA (TARGET_FILE_SIZE_MB=128);
OPTIMIZE TABLE 'employee' FOR PARTITIONS year=2023;
```

**Rewrite Manifests:**
```sql
OPTIMIZE TABLE 'employee' REWRITE MANIFESTS;
```

---
