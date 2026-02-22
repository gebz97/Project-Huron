
## Chapter 14: Data Sharing with Delta Sharing Protocol

### The Problem with Traditional Data Sharing

**Issues:**
- Copying data to multiple locations
- 40+ separate exports = complexity
- Distributed synchronization problems
- Egress/ingress costs
- Multiple sources of truth

### Delta Sharing Protocol

**Open solution for secure, live data sharing**

```
Data Provider                 Data Recipient
    ↓                              ↓
   Share → Schema → Table    Profile + Bearer Token
    ↓                              ↓
  Access Controls               Direct Query
```

### Data Providers

**Share configuration (YAML):**
```yaml
version: 1
shares:
- name: "consumer_marketing_analysts_secure-read"
  schemas:
  - name: "consumer"
    tables:
    - name: "clickstream_hourly"
      location: "s3a://.../common_foods/consumer/clickstream_hourly"
      id: "eb6f82f5-a738-4bd8-943c-9cd8594b12ac"
```

### Data Recipients

**Recipient profile (JSON):**
```json
{
  "shareCredentialsVersion": 1,
  "endpoint": "https://commonfoods.io/delta-sharing/",
  "bearerToken": "<token>",
  "expirationTime": "2023-08-11T00:00:00.0Z"
}
```

### REST API

**List shares:**
```bash
curl -XGET \
  --header 'Authorization: Bearer $BEARER_TOKEN' \
  --url "$DELTA_SHARING_ENDPOINT/shares?maxResults=10"
```

Response:
```json
{
  "items": [{"name":"delta_sharing"}]
}
```

**Get share:**
```bash
curl -XGET \
  --header 'Authorization: Bearer $BEARER_TOKEN' \
  --url "$DELTA_SHARING_ENDPOINT/shares/delta_sharing"
```

**List schemas in share:**
```bash
curl -XGET \
  --header 'Authorization: Bearer $BEARER_TOKEN' \
  --url "$DELTA_SHARING_ENDPOINT/shares/delta_sharing/schemas"
```

**List tables in schema:**
```bash
curl -XGET \
  --header 'Authorization: Bearer $BEARER_TOKEN' \
  --url "$DELTA_SHARING_ENDPOINT/shares/delta_sharing/schemas/default/tables?maxResults=4"
```

Response:
```json
{
  "items": [
    {"name":"COVID_19_NYT","schema":"default","share":"delta_sharing"},
    {"name":"boston-housing","schema":"default","share":"delta_sharing"}
  ],
  "nextPageToken": "CgE0Eg1kZWx0YV9zaGFyaW5nGgdkZWZhdWx0"
}
```

**List all tables in share:**
```bash
curl -XGET \
  --header 'Authorization: Bearer $BEARER_TOKEN' \
  --url "$DELTA_SHARING_ENDPOINT/shares/delta_sharing/all-tables"
```

### Delta Sharing with PySpark

**Install:**
```bash
pip install delta-sharing
```

**SparkSession with delta-sharing JAR:**
```python
spark = SparkSession.builder \
  .master("local[*]") \
  .config("spark.jars.packages", "io.delta:delta-sharing-spark_2.12:3.1.0") \
  .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
  .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
  .appName("delta_sharing_dldg") \
  .getOrCreate()
```

**Generate table URL:**
```python
from delta_sharing import SharingClient

profile_path = "..."
client = SharingClient(f"{profile_path}/open-datasets.share")

shares = client.list_shares()
first_share = shares[0]
schemas = client.list_schemas(first_share)
first_schema = schemas[0]
tables = client.list_tables(first_schema)
table = tables[3]  # lending_club

table_url = f"{profile_path}/open-datasets.share#{table.share}.{table.schema}.{table.name}"
```

**Read remote table:**
```python
df = (spark.read
 .format("deltaSharing")
 .option("responseFormat", "parquet")
 .option("startingVersion", 1)
 .load(table_url)
 .select("loan_amnt", "funded_amnt", "term", "grade"))
```

### Delta Sharing with Spark Scala

```scala
import org.apache.spark.sql.functions.{col}

val profile_file_location = "/path/to/open-datasets.share"
val table_url = s"$profile_file_location#delta_sharing.default.owid-covid-data"

val df = spark.read
  .format("deltaSharing")
  .load(table_url)
  .select("iso_code", "location", "date")
  .where(col("iso_code").equalTo("USA"))
  .limit(100)
```

### Delta Sharing with Spark SQL

```sql
CREATE TABLE lending_club USING deltaSharing 
LOCATION '<profile-file-path>#delta_sharing.default.lending_club';

SELECT * FROM lending_club;
```

### Delta Sharing Options

| Option | Type | Description |
|--------|------|-------------|
| `readChangeFeed` | Boolean | Read change data feed |
| `maxVersionsPerRpc` | String | Control volume per RPC |
| `startingVersion` | Int | Time travel starting version |
| `endingVersion` | Int | Bound read range |
| `startingTimestamp` | Timestamp | Read from closest transaction |
| `endingTimestamp` | Timestamp | Bound to timestamp |
| `responseFormat` | String | "delta" or "parquet" |
| `maxFilesPerTrigger` | Int | Files per microbatch |
| `maxBytesPerTrigger` | String | Approximate bytes per batch |
| `ignoreChanges` | Boolean | Reprocess updates |
| `ignoreDeletes` | Boolean | Ignore partition deletes |
| `skipChangeCommits` | Boolean | Ignore delete/modify transactions |

### Streaming with Delta Shares

```scala
val df = spark.readStream.format("deltaSharing")
  .option("startingVersion", "1")
  .option("skipChangeCommits", "true")
  .load(tablePath)
```

### Community Connectors

| Connector | Status |
|-----------|--------|
| Power BI | Released |
| Node.js | Released |
| Java | Released |
| Arcuate | Released |
| Rust | Released |
| Go | Released |
| C++ | Released |

---
