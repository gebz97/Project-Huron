
## Chapter 13: Migrating to Apache Iceberg

### Migration Considerations

| Factor | Consideration |
|--------|---------------|
| Data structures | Restructure to Iceberg schema |
| Pipelines | Update ETL to write Iceberg |
| Workflows | Adjust dependencies and scheduling |

### In-Place vs. Shadow Migration

| Approach | Pros | Cons |
|----------|------|------|
| **In-place** | Simple, lower storage cost | Riskier, no easy rollback |
| **Shadow** | Safe, can test, preserve original | Complex, higher storage |

### Three-Step In-Place Migration

1. Check record/file count in old table partition
2. Migrate partition to Iceberg
3. Verify counts match

### Four-Phase Shadow Migration

- **Phase 1:** Write to old, read from old (setup Iceberg)
- **Phase 2:** Write to both, read from old (dual-write)
- **Phase 3:** Write to both, read from new (test reads)
- **Phase 4:** Write to new, read from new (cutover)

### Migrating Hive Tables

**snapshot() - Create temporary copy:**
```sql
CALL catalog.system.snapshot('hive.db.tableA', 'db.tableAsnapshot');
CALL catalog.system.snapshot('hive.db.tableA', 'db.tableAsnapshot', 's3://bucket/location');
```

**migrate() - Permanently migrate:**
```sql
CALL catalog.system.migrate('hive.db.sample', map('foo', 'bar'));
```

### Migrating Delta Lake

```java
import org.apache.iceberg.catalog.TableIdentifier;
import org.apache.iceberg.delta.DeltaLakeToIcebergMigrationActionsProvider;

DeltaLakeToIcebergMigrationActionsProvider.defaultActions()
  .snapshotDeltaLakeTable("s3://my-bucket/delta-table")
  .as(TableIdentifier.parse("my_db.my_table"))
  .icebergCatalog(icebergCatalog)
  .tableLocation("s3://my-bucket/iceberg-table")
  .deltaLakeConfiguration(hadoopConf)
  .tableProperty("my_property", "my_value")
  .execute();
```

### Migrating Apache Hudi

```java
import org.apache.iceberg.hudi.HudiToIcebergMigrationActionsProvider;

HudiToIcebergMigrationActionsProvider.defaultProvider()
  .snapshotHudiTable("hdfs://my-hudi-table")
  .as(TableIdentifier.parse("my_db.my_table"))
  .hoodieConfiguration(hadoopConf)
  .icebergCatalog(icebergCatalog)
  .execute();
```

### Migrating Individual Files

**add_files procedure:**
```sql
CALL catalog.system.add_files(
  table => 'db.my_table',
  source_table => 's3://my-parquet-tables/tables',
  partition_filter => map('partition_col', 'partition_value'),
  check_duplicate_files => true
);
```

### Migrating by Rewriting Data

**CTAS - New table:**
```sql
CREATE TABLE catalog.db.tableA
USING iceberg
PARTITIONED BY (month(ts_field))
AS
SELECT *,
  CAST(old_field AS <data_type>) AS updated_field
FROM my_source_table;
```

**COPY INTO - Existing table (Dremio):**
```sql
COPY INTO catalog.db.my_iceberg_table
FROM '@my_dremio_source/folder'
FILE_FORMAT 'csv';

-- Incremental with regex
COPY INTO my_table
FROM '@my_storage_location'
REGEX '^2023-10-11_.*\.csv'
FILE_FORMAT 'csv';
```

**INSERT INTO SELECT:**
```sql
INSERT INTO tableB SELECT * FROM tableA;

-- MERGE for incremental updates
MERGE INTO tableB AS target
USING (
  SELECT * FROM tableA
  WHERE created_at >= DATE_DIFF(CURRENT_DATE(), INTERVAL 30 DAY)
  OR updated_at >= DATE_DIFF(CURRENT_DATE(), INTERVAL 30 DAY)
) AS source ON target.id = source.id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT *;
```

---
