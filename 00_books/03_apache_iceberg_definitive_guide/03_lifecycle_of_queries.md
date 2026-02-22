
## Chapter 3: Lifecycle of Write and Read Queries

### Creating a Table

```sql
-- Spark SQL
CREATE TABLE orders (
  order_id BIGINT,
  customer_id BIGINT,
  order_amount DECIMAL(10,2),
  order_ts TIMESTAMP
)
USING iceberg
PARTITIONED BY (HOUR(order_ts));
```

**What Happens:**
1. Engine parses query
2. Creates metadata file `v1.metadata.json`
3. Writes schema and partition spec
4. Updates catalog pointer to `v1.metadata.json`

**Metadata File Contents:**
```json
{
  "table-uuid": "072db680-d810-49ac-935c-56e901cad686",
  "schema": {
    "type": "struct",
    "fields": [
      {"id": 1, "name": "order_id", "type": "long"},
      {"id": 2, "name": "customer_id", "type": "long"},
      {"id": 3, "name": "order_amount", "type": "decimal(10,2)"},
      {"id": 4, "name": "order_ts", "type": "timestamp"}
    ]
  },
  "partition-spec": [
    {"name": "order_ts_hour", "transform": "hour", "source-id": 4}
  ]
}
```

### Insert Query

```sql
INSERT INTO orders VALUES 
  (123, 456, 36.17, '2023-03-07 08:10:23');
```

**Step-by-Step Process:**

1. **Check Catalog** → Get current metadata file (`v1.metadata.json`)
2. **Write Datafile** → 
   ```
   s3://datalake/db1/orders/data/order_ts_hour=2023-03-07-08/0_0_0.parquet
   ```
3. **Write Manifest File** → `62acb3d7-e992-4cbc-8e41-58809fcacb3e.avro`
   - Lists datafile + statistics
4. **Write Manifest List** → `snap-8333017788700497002-1-4010cc03...avro`
   - Lists manifest file + partition stats
5. **Write Metadata File** → `v2.metadata.json`
   - New snapshot `s1` with manifest list
6. **Update Catalog** → Point to `v2.metadata.json`

### Merge Query (UPSERT)

```sql
MERGE INTO orders o
USING orders_staging s ON o.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET order_amount = s.order_amount
WHEN NOT MATCHED THEN INSERT *;
```

**Copy-on-Write (COW) Behavior:**
- Read matching records into memory
- Create new datafile with updated values
- Old file remains (for historical snapshots)

**Resulting Files:**
```
data/order_ts_hour=2023-03-07-08/0_0_0.parquet (old)
data/order_ts_hour=2023-03-07-08/0_0_1.parquet (updated)
data/order_ts_hour=2023-01-27-10/0_0_0.parquet (new insert)
```

### Read Query

```sql
SELECT * FROM orders 
WHERE order_ts BETWEEN '2023-01-01' AND '2023-01-31';
```

**Step-by-Step Process:**

1. **Check Catalog** → Get current metadata (`v3.metadata.json`)
2. **Read Metadata File** → Get `current-snapshot-id` and manifest list path
3. **Read Manifest List** → Get manifest file paths + partition stats
4. **Prune Manifests** → Skip manifests whose partitions don't match filter
5. **Read Manifest Files** → Get datafile paths + column stats
6. **Prune Datafiles** → Skip files whose min/max don't match filter
7. **Read Datafiles** → Return matching records

### Time-Travel Query

```sql
-- By timestamp
SELECT * FROM orders TIMESTAMP AS OF '2023-03-07 20:45:08.914';

-- By snapshot ID
SELECT * FROM orders VERSION AS OF 8333017788700497002;
```

**Process:**
1. Get current metadata file
2. Find snapshot matching timestamp/ID
3. Use that snapshot's manifest list
4. Read data as of that snapshot

---
