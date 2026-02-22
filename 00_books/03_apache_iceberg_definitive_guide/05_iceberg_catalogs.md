
## Chapter 5: Iceberg Catalogs

### Catalog Requirements

**Minimum:**
- List tables
- Create/drop tables
- Check existence
- Rename tables
- Map table path → metadata file location

**Production Requirement:**
- Atomic updates to metadata pointer
- Prevents data loss with concurrent writers

### Catalog Comparison

| Catalog | Production Ready | External System | Engine Support | Multi-table Transactions | Cloud Agnostic |
|---------|------------------|-----------------|----------------|-------------------------|----------------|
| **Hadoop** | No (without DynamoDB) | No (filesystem) | High | No | Yes |
| **Hive** | Yes | Self-hosted | High | No | Yes |
| **AWS Glue** | Yes | Managed | AWS services | No | No |
| **Nessie** | Yes | Self-hosted | Growing | Yes | Yes |
| **REST** | Yes | Custom service | Growing | Yes | Yes |
| **JDBC** | Yes | Self-hosted | Medium | No | Yes |

### Hadoop Catalog

**How it works:** `version-hint.txt` in metadata folder contains version number

**Pros:**
- No external system needed
- Easy to start

**Cons:**
- Not production-safe (no atomic renames on S3)
- Single warehouse directory only
- Slow namespace listing
- Can't DROP without deleting data

**Configure Spark:**
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:0.14.0 \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_catalog.type=hadoop \
  --conf spark.sql.catalog.my_catalog.warehouse=s3://my-bucket/warehouse/
```

### Hive Catalog

**How it works:** `location` table property in Hive Metastore

**Pros:**
- Wide engine compatibility
- Cloud agnostic

**Cons:**
- Must self-host Hive Metastore
- No multi-table transactions

**Configure Spark:**
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:0.14.0 \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_catalog.type=hive \
  --conf spark.sql.catalog.my_catalog.uri=thrift://metastore-host:9083
```

### AWS Glue Catalog

**How it works:** `metadata_location` table property in Glue

**Pros:**
- Managed service
- Tight AWS integration

**Cons:**
- AWS-specific
- No multi-table transactions

**Configure Spark:**
```bash
spark-sql --packages "org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:0.14.0,software.amazon.awssdk:bundle:2.17.178" \
  --conf spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_catalog.warehouse=s3://my-bucket/warehouse/ \
  --conf spark.sql.catalog.my_catalog.catalog-impl=org.apache.iceberg.aws.glue.GlueCatalog \
  --conf spark.sql.catalog.my_catalog.io-impl=org.apache.iceberg.aws.s3.S3FileIO
```

### Nessie Catalog

**How it works:** `metadataLocation` table property in Nessie

**Pros:**
- Git-like experience ("data as code")
- Multi-table transactions
- Cloud agnostic

**Cons:**
- Self-host (or use Dremio Arctic)
- Limited engine support

**Configure Spark:**
```bash
spark-sql --packages "org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:0.14.0,org.projectnessie:nessie-spark-extensions-3.3_2.12:0.XX.X" \
  --conf spark.sql.extensions="org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,org.projectnessie.spark.extensions.NessieSparkSessionExtensions" \
  --conf spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_catalog.catalog-impl=org.apache.iceberg.nessie.NessieCatalog \
  --conf spark.sql.catalog.my_catalog.uri=http://localhost:19120/api/v1 \
  --conf spark.sql.catalog.my_catalog.ref=main \
  --conf spark.sql.catalog.my_catalog.warehouse=s3://my-bucket/warehouse/
```

### REST Catalog

**How it works:** RESTful service maps table → metadata

**Pros:**
- Fewer dependencies
- Flexible backend
- Multi-table transactions
- Cloud agnostic

**Cons:**
- Must implement or use hosted service
- Limited engine support

**Configure Spark:**
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:0.14.0 \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_catalog.type=rest \
  --conf spark.sql.catalog.my_catalog.uri=http://rest-catalog:8080
```

### JDBC Catalog

**How it works:** `metadata_location` in JDBC-compliant database

**Pros:**
- Easy if you have PostgreSQL/MySQL
- Cloud-hosted options (RDS, Azure Database)
- Cloud agnostic

**Cons:**
- No multi-table transactions
- Need JDBC driver in classpath

**Configure Spark:**
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:0.14.0,mysql:mysql-connector-java:8.0.23 \
  --conf spark.sql.catalog.my_catalog=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.my_catalog.warehouse=s3://my-bucket/warehouse/ \
  --conf spark.sql.catalog.my_catalog.catalog-impl=org.apache.iceberg.jdbc.JdbcCatalog \
  --conf spark.sql.catalog.my_catalog.uri=jdbc:mysql://localhost:3306/iceberg \
  --conf spark.sql.catalog.my_catalog.jdbc.user=username \
  --conf spark.sql.catalog.my_catalog.jdbc.password=password
```

### Catalog Migration

**Two Methods:**

1. **Iceberg Catalog Migration CLI:**
```bash
java -jar iceberg-catalog-migrator-cli-0.2.0.jar migrate \
  --source-catalog-type GLUE \
  --source-catalog-properties warehouse=s3://bucket/gluecatalog/ \
  --target-catalog-type NESSIE \
  --target-catalog-properties uri=http://localhost:19120/api/v1,warehouse=s3://bucket/nessie/ \
  --identifiers db1.table1
```

2. **Using Spark SQL:**

**register_table() - Lightweight copy:**
```sql
CALL target_catalog.system.register_table(
  'target_catalog.db.table1', 
  '/path/to/source/table/metadata/xxx.json'
);
```

**snapshot() - Copy with new location:**
```sql
CALL target_catalog.system.snapshot(
  'source_catalog.db.table1',
  'target_catalog.db.table1'
);
```

---
