
## Chapter 14: Real-World Use Cases

### Write-Audit-Publish (WAP) Pattern

**1. Create Branch:**
```sql
ALTER TABLE catalog.db.table CREATE BRANCH etl_branch;
```

**2. Enable WAP:**
```sql
ALTER TABLE catalog.db.table SET TBLPROPERTIES ('write.wap.enabled'='true');
spark.conf.set('spark.wap.branch', 'etl_branch');
```

**3. Write to Branch:**
```sql
INSERT INTO catalog.db.table SELECT * FROM new_data;
```

**4. Audit Data:**
```python
# Check for nulls
null_counts = df.select([count(when(col(c).isNull(), c)).alias(c) for c in df.columns])

# Check duplicates
duplicates = df.groupBy(df.columns).count().filter(col("count") > 1)

# Date range check
within_range = df.filter((col("date_column") >= start_date) & (col("date_column") <= end_date))
count_within_range = within_range.count()
```

**5. Fix Issues:**
```python
# Identify problematic records
null_condition = [col(c).isNull() for c in df.columns]
combined_null = null_condition[0]
for cond in null_condition[1:]:
    combined_null = combined_null | cond

duplicates_cond = df.groupBy(df.columns).count().filter(col("count") > 1).drop("count")

records_needing_fix = df.filter(combined_null).union(df.join(duplicates_cond, df.columns, "inner")).distinct()
valid_records = df.exceptAll(records_needing_fix)

# Overwrite branch with clean data
valid_records.write.format("iceberg").mode("overwrite").save("catalog.db.table")
```

**6. Publish to Main:**
```sql
-- Find snapshot ID
SELECT * FROM catalog.db.table.refs;

-- Cherry-pick snapshot
CALL catalog.system.cherrypick_snapshot('db.table', 2668401536062194692);

-- Turn off WAP
spark.conf.unset('spark.wap.branch');
```

### BI Workloads on Data Lake

**1. Land Data as Iceberg Tables**

**2. Create Virtual Data Marts (views)**

**3. Create Aggregate Reflection (Dremio):**
```sql
ALTER VIEW oh_shipments
CREATE AGGREGATE REFLECTION oh_shipments_agg
USING
DIMENSIONS (shipper_id, destination_city, shipping_method)
MEASURES (shipment_id (COUNT), total_cost (SUM), delivery_time (AVG))
LOCALSORT BY (shipper_id, destination_city);
```

**4. Connect BI Tool to Dremio**

### Change Data Capture (CDC)

**1. Create Tables:**
```sql
CREATE TABLE glue.test.inventory (
  product_id INT,
  product_name STRING,
  stock_level INT,
  price INT,
  last_updated DATE
) USING iceberg;

INSERT INTO glue.test.inventory VALUES 
  (1, 'Pasta-thin', 60, 45, '3/25/2023'),
  (2, 'Bread-white', 55, 6, '3/10/2023'),
  (3, 'Eggs-nonorg', 100, 8, '3/12/2023');

CREATE TABLE glue.test.inventory_summary (
  product_id INT,
  total_stock INT,
  avg_price DOUBLE
) USING iceberg;

INSERT INTO glue.test.inventory_summary
SELECT product_id, SUM(stock_level) AS total_stock, AVG(price) AS avg_price
FROM glue.test.inventory
GROUP BY product_id;
```

**2. Apply Updates:**
```sql
UPDATE glue.test.inventory
SET stock_level = stock_level - 15
WHERE product_name = 'Bread-white';
```

**3. Create Change Log View:**
```sql
-- Find snapshot IDs
SELECT * FROM glue.test.inventory.history;

-- Create changelog view
CALL glue.system.create_changelog_view(
  table => 'glue.test.inventory',
  options => map(
    'start-snapshot-id', '4816648710583642722',
    'end-snapshot-id', '2557325773776943708'
  )
);
```

**4. Merge Changes to Summary:**
```sql
-- Create aggregated changes view
CREATE OR REPLACE TEMPORARY VIEW aggregated_changes AS
SELECT
  product_id,
  SUM(CASE 
    WHEN _change_type = 'INSERT' THEN stock_level
    WHEN _change_type = 'DELETE' THEN -stock_level
    ELSE 0 END) AS total_stock_change,
  AVG(price) AS new_avg_price
FROM inventory_changes
GROUP BY product_id;

-- Merge into summary
MERGE INTO glue.test.inventory_summary AS target
USING aggregated_changes AS source ON target.product_id = source.product_id
WHEN MATCHED THEN 
  UPDATE SET 
    target.total_stock = target.total_stock + source.total_stock_change
WHEN NOT MATCHED THEN 
  INSERT (product_id, total_stock, avg_price)
  VALUES (source.product_id, source.total_stock_change, source.new_avg_price);
```

---
