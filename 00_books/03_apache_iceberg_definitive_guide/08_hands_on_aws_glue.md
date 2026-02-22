
## Chapter 8: AWS Glue

### Configuration

**Create Glue Database:**
- Navigate to AWS Glue Console → Data Catalog → Databases
- Click "Add database"
- Enter name (e.g., "test")

**Configure Glue ETL Job:**

1. **Visual ETL** → Visual with Blank Canvas
2. **Add Node** → Amazon S3 (source)
3. **Data Source Properties:** Set S3 path and CSV format
4. **Job Details Tab:**

| Setting | Value |
|---------|-------|
| Name | Iceberg_ETL |
| IAM Role | Appropriate permissions |
| Glue Version | Glue 4.0 |
| Script language | Python 3 |
| Worker type | Standard |
| Requested number of workers | 2 |

5. **Advanced Properties → Job Parameters:**
   - Key: `--datalake-formats` → Value: `iceberg`
   - Key: `--conf` → Value: `spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions --conf spark.sql.catalog.glue_catalog=org.apache.iceberg.spark.SparkCatalog --conf spark.sql.catalog.glue_catalog.warehouse=s3://your-warehouse-dir/ --conf spark.sql.catalog.glue_catalog.catalog-impl=org.apache.iceberg.aws.glue.GlueCatalog --conf spark.sql.catalog.glue_catalog.io-impl=org.apache.iceberg.aws.s3.S3FileIO`

### Generated Script (with modifications)

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Read S3 data as DataFrame
df = glueContext.create_data_frame.from_options(
    connection_type="s3",
    format="csv",
    connection_options={
        "paths": ["s3://dm-iceberg/datasets/salesdata.csv"]
    }
).toDF()

# Create temp view
df.createOrReplaceTempView("temp_df")

# Create Iceberg table
query = """
CREATE TABLE glue_catalog.test.employee
USING iceberg
AS SELECT * FROM temp_df
"""
spark.sql(query)

job.commit()
```

### Operations

**Read Table:**
```python
df = glueContext.create_data_frame.from_catalog(
    database="test",
    table_name="employee"
)
```

**Insert Data:**
```python
glueContext.write_data_frame.from_catalog(
    frame=df,
    database="test",
    table_name="employee"
)
```

---
