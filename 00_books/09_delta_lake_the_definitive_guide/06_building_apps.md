
## Chapter 6: Building Native Applications with Delta Lake

### Python with deltalake

**Installation:**
```bash
pip install 'deltalake>=0.18.2' pandas
```

**Basic Usage:**
```python
from deltalake import DeltaTable

dt = DeltaTable('./deltatbl-partitioned')
print(dt.files())
df = dt.to_pandas()
print(df)
```

### Reading Large Datasets

**Optimizations:**
1. **Partitions** - Group files by common prefixes
2. **File statistics** - Min/max values in transaction log

**Partition Filtering:**
```python
# Load only partition 'foo0'
df = dt.to_pandas(partitions=[('c2', '=', 'foo0')])
print(df)

# Preview files that would be loaded
print(dt.files([('c2', '=', 'foo0')]))
```

**Column Projection:**
```python
df = dt.to_pandas(
  partitions=[('c2', '=', 'foo0')],
  columns=['c1']
)
```

**Filter Predicates:**
```python
df = dt.to_pandas(
  partitions=[('c2', '=', 'foo0')],
  columns=['c1'],
  filters=[('c1', '<=', 4), ('c1', '>', 0)]
)
```

### File Statistics

**Transaction log entry with stats:**
```json
{
  "add": {
    "path": "year=2022/0-ec9935aa-a154-4ba4-ab7e-92a53369c433-2.parquet",
    "stats": "{\"numRecords\":4,\"minValues\":{\"month\":9},\"maxValues\":{\"month\":12}}"
  }
}
```

**Using stats for efficient query:**
```python
df = dt.to_pandas(filters=[
  ('year', '=', 2022),
  ('month', '>=', 9)
])
```

### Writing Data

```python
import pandas as pd
from deltalake import write_deltalake, DeltaTable

df = pd.read_csv('./data/co2_mm_mlo.csv', comment='#')
write_deltalake('./data/co2_monthly', df)

dt = DeltaTable('./data/co2_monthly')
print(dt.files())
```

**Partitioned Write:**
```python
write_deltalake(
  './data/gen/co2_monthly_partitioned',
  data=df,
  partition_by=['year']
)
```

**Write Modes:**
- `error` (default) - Error if table exists
- `append` - Add data to table
- `overwrite` - Replace table contents
- `ignore` - Don't write if exists

### Merge/Update

```python
import pyarrow as pa

data = pa.table({'id': list(range(100))})
write_deltalake('delete-test', data)

dt = DeltaTable('delete-test')
print(dt.version())  # 0

# Delete every other row
dt.delete('id % 2 == 0')
print(dt.version())  # 1
```

**Transaction log shows:**
- `remove` - Remove old file
- `add` - Add new file with remaining rows

### PyArrow Integration

**DataSet (lazy loading):**
```python
dataset = dt.to_pyarrow_dataset()
filtered = dataset.filter(("year", "=", 2022))
for batch in filtered.to_batches():
    process(batch)
```

**Table (eager loading):**
```python
table = dt.to_pyarrow_table(
  partitions=[('year', '=', 2022)],
  filters=[('month', '>=', 9)]
)
```

### Rust with delta-rs

**Cargo.toml:**
```toml
[dependencies]
deltalake = { version = "0.19", features = ["datafusion"] }
tokio = { version = "1", features = ["full"] }
```

**Open Table:**
```rust
use deltalake::open_table;

#[tokio::main]
async fn main() {
    let table = open_table("./data/deltatbl-partitioned")
        .await
        .expect("Failed to open table");
    
    println!("Loaded version {}", table.version());
    for file in table.get_files_iter() {
        println!(" - {}", file.as_ref());
    }
}
```

**Query with DataFusion:**
```rust
use std::sync::Arc;
use deltalake::datafusion::execution::context::SessionContext;

#[tokio::main]
async fn main() {
    let ctx = SessionContext::new();
    let table = open_table("./data/deltatbl-partitioned").await.unwrap();
    ctx.register_table("demo", Arc::new(table)).unwrap();

    let df = ctx.sql("SELECT * FROM demo LIMIT 5").await.unwrap();
    df.show().await.unwrap();
}
```

**Partitioned Query:**
```rust
let df = ctx.sql("SELECT * FROM demo WHERE c2 = 'foo0'")
    .await
    .expect("Failed to create data frame");
```

**Delete Operation:**
```rust
use deltalake::operations::delete::DeleteBuilder;
use deltalake::DeltaOps;

let table = open_table("./test").await?;
let (table, metrics) = DeltaOps::from(table)
    .delete()
    .with_predicate(col("id").mod(2).eq(lit(0)))
    .await?;
```

### AWS Lambda with Python

**Basic example:**
```python
import os
from deltalake import DeltaTable

def lambda_handler(event, context):
    url = os.environ['TABLE_URL']
    dt = DeltaTable(url)
    return {
        'version': dt.version(),
        'table': url,
        'files': dt.files()
    }
```

**Concurrent writes on S3:**
```
Error: Atomic rename requires a LockClient for S3 backends.
```

**Solutions:**
1. **S3DynamoDBLogStore** (recommended) - Uses DynamoDB for coordination
2. **AWS_S3_ALLOW_UNSAFE_RENAME=true** (risk of corruption)

### Rust Lambda

**Setup:**
```bash
cargo lambda new deltadog --event-type s3::S3Event
cd deltadog
cargo add --features s3 deltalake
cargo lambda build --release --output-format zip
```

**Function handler:**
```rust
async fn function_handler(event: LambdaEvent<S3Event>) -> Result<(), Error> {
    let table = deltalake::open_table("s3://example/table").await?;
    Ok(())
}
```

---
