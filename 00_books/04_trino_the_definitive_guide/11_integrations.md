
## Chapter 11: Integrating Trino with Other Tools

### Apache Superset

- Web-based BI tool
- SQL Lab editor with metadata browsing
- Rich visualizations and dashboards
- Connect via Trino JDBC driver

### RubiX (Caching)

- Lightweight data-caching framework
- Disk and in-memory caching
- Built into Hive connector
- Configure storage caching on workers

### Apache Airflow

- Workflow orchestration
- Trino hooks available
- Run Trino CLI from shell scripts
- Orchestrate data pipelines

### Amazon Athena

**Serverless Trino-based service:**
- Pay per query (data scanned)
- No infrastructure management
- Integrates with S3 and Glue Catalog
- JDBC/ODBC connectivity

**Example:**
```bash
# Run query
aws athena start-query-execution \
  --cli-input-json file://athena-input.json \
  --query-string 'SELECT species, AVG(petal_length_cm) FROM iris GROUP BY species'

# Check status
aws athena get-query-execution --query-execution-id <id>

# Get results from S3
aws s3 cp s3://bucket/results/<id>.csv /dev/stdout
```

### Starburst Enterprise

- Commercial Trino distribution
- Additional connectors
- Management console
- Enhanced security features
- Enterprise support

---
