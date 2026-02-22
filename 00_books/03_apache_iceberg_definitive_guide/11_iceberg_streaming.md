
## Chapter 11: Streaming with Apache Iceberg

### Streaming with Spark

**Streaming into Iceberg:**
```scala
val spark = SparkSession.builder()
  .appName("FinancialDataStreaming")
  .getOrCreate()

// Read from Kafka
val df = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "financialData")
  .load()

// Parse JSON
val financialData = df.selectExpr("CAST(value AS STRING)").as[String]
  .map(Stock.from_json(_))

// Write to Iceberg (append)
val tableIdentifier = "s3://bucket/financial_data/stock"
financialData.writeStream
  .format("iceberg")
  .outputMode("append")
  .trigger(Trigger.ProcessingTime(1, TimeUnit.MINUTES))
  .option("path", tableIdentifier)
  .option("checkpointLocation", "/tmp/checkpoints")
  .start()

// For partitioned tables, enable fanout
financialData.writeStream
  .format("iceberg")
  .outputMode("append")
  .option("fanout-enabled", "true")
  .option("checkpointLocation", "/tmp/checkpoints")
  .start()
```

**Streaming from Iceberg:**
```scala
val df = spark.readStream
  .format("iceberg")
  .option("stream-from-timestamp", streamStartTimestamp.toString)
  .load(tableName)

df.writeStream
  .format("iceberg")
  .outputMode("append")
  .trigger(Trigger.ProcessingTime(1, TimeUnit.MINUTES))
  .option("path", destinationTableName)
  .option("checkpointLocation", checkpointPath)
  .start()
```

### Streaming with Flink

**Flink SQL - Batch Read:**
```sql
SET execution.runtime-mode = batch;
SELECT * FROM sales_data;
```

**Flink SQL - Streaming Read:**
```sql
SET execution.runtime-mode = streaming;
SET table.dynamic-table-options.enabled = true;
SELECT * FROM sales_data /*+ OPTIONS('streaming'='true', 'monitor-interval'='1s')*/;
```

**Flink SQL - Read from snapshot:**
```sql
SELECT * FROM sales_data /*+ OPTIONS(
  'streaming'='true', 
  'monitor-interval'='1s', 
  'start-snapshot-id'='3821550127947089987'
)*/;
```

**Flink DataStream API - Batch Read:**
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
TableLoader tableLoader = TableLoader.fromHadoopTable("s3://bucket/retail/sales_data");

IcebergSource<RowData> source = IcebergSource.forRowData()
  .tableLoader(tableLoader)
  .assignerFactory(new SimpleSplitAssignerFactory())
  .build();

DataStream<RowData> batch = env.fromSource(
  source,
  WatermarkStrategy.noWatermarks(),
  "Iceberg Source"
);
```

**Flink DataStream API - Streaming Read:**
```java
IcebergSource<RowData> source = IcebergSource.forRowData()
  .tableLoader(tableLoader)
  .assignerFactory(new SimpleSplitAssignerFactory())
  .streaming(true)
  .streamingStartingStrategy(StreamingStartingStrategy.INCREMENTAL_FROM_LATEST_SNAPSHOT)
  .monitorInterval(Duration.ofSeconds(60))
  .build();
```

**Flink DataStream API - Write:**
```java
DataStream<RowData> input = ...;
FlinkSink.forRowData(input)
  .tableLoader(tableLoader)
  .append();
```

**Flink DataStream API - Overwrite:**
```java
FlinkSink.forRowData(input)
  .tableLoader(tableLoader)
  .overwrite(true)
  .append();
```

**Flink DataStream API - Upsert:**
```java
FlinkSink.forRowData(input)
  .tableLoader(tableLoader)
  .upsert(true)
  .append();
```

### Kafka Connect with Iceberg

**Worker Properties (worker.properties):**
```properties
bootstrap.servers=kafka-broker1:9092,kafka-broker2:9092
connector.class=io.tabular.iceberg.connect.IcebergSinkConnector
tasks.max=2
topics=events
iceberg.tables=default.events
iceberg.catalog.type=rest
iceberg.catalog.uri=http://localhost
iceberg.catalog.warehouse=my_warehouse
```

**Start Kafka Connect:**
```bash
bin/connect-distributed.sh config/worker.properties
```

---
