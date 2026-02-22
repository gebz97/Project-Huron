
## Chapter 11: Managing and Deploying ML Pipelines

### MLflow Components

```
┌─────────────────────────────────────┐
│           MLflow                    │
├──────────┬──────────┬──────────────┤
│ Tracking │ Projects │   Models     │
├──────────┴──────────┴──────────────┤
│            Registry                 │
└─────────────────────────────────────┘
```

### MLflow Tracking

**Install MLflow:**
```bash
pip install mlflow
```

**Track Random Forest Experiment:**
```python
import mlflow
import mlflow.spark
import pandas as pd

with mlflow.start_run(run_name="random-forest"):
    # Log parameters
    mlflow.log_param("num_trees", rf.getNumTrees())
    mlflow.log_param("max_depth", rf.getMaxDepth())
    
    # Train model
    pipelineModel = pipeline.fit(trainDF)
    
    # Log model
    mlflow.spark.log_model(pipelineModel, "model")
    
    # Evaluate
    predDF = pipelineModel.transform(testDF)
    rmse = evaluator.setMetricName("rmse").evaluate(predDF)
    r2 = evaluator.setMetricName("r2").evaluate(predDF)
    
    # Log metrics
    mlflow.log_metrics({"rmse": rmse, "r2": r2})
    
    # Log artifact
    rfModel = pipelineModel.stages[-1]
    pandasDF = pd.DataFrame(
        list(zip(vecAssembler.getInputCols(), rfModel.featureImportances)),
        columns=["feature", "importance"]
    ).sort_values("importance", ascending=False)
    
    pandasDF.to_csv("feature-importance.csv", index=False)
    mlflow.log_artifact("feature-importance.csv")
```

**Start MLflow UI:**
```bash
mlflow ui
# Navigate to http://localhost:5000
```

**Query MLflow Tracking Server:**
```python
from mlflow.tracking import MlflowClient

client = MlflowClient()
runs = client.search_runs(run.info.experiment_id, 
                          order_by=["attributes.start_time desc"], 
                          max_results=1)
run_id = runs[0].info.run_id
print(runs[0].data.metrics)
```

### MLflow Projects

**project.yaml:**
```yaml
name: RandomForestExample
conda_env: conda.yaml

entry_points:
  main:
    parameters:
      max_depth: {type: int, default: 5}
      num_trees: {type: int, default: 100}
    command: "python train.py --max_depth {max_depth} --num_trees {num_trees}"
```

**Run MLflow Project:**
```python
mlflow.run("https://github.com/user/repo/#project-name", 
           parameters={"max_depth": 5, "num_trees": 100})
```

### Deployment Options Comparison

| Option | Throughput | Latency | Example |
|--------|------------|---------|---------|
| **Batch** | High | Hours-days | Customer churn prediction |
| **Streaming** | Medium | Seconds-minutes | Dynamic pricing |
| **Real-time** | Low | Milliseconds | Online ad bidding |

### Batch Deployment

```python
# Load saved model
import mlflow.spark
pipelineModel = mlflow.spark.load_model(f"runs:/{run_id}/model")

# Generate predictions
inputDF = spark.read.parquet("/path/to/data")
predDF = pipelineModel.transform(inputDF)

# Save results
predDF.write.mode("overwrite").save("/path/to/predictions")
```

### MLflow Model Registry

```python
# Register model
mlflow.register_model(f"runs:/{run_id}/model", "AirbnbPriceModel")

# Load production model
model_production_uri = "models:/AirbnbPriceModel/production"
model_production = mlflow.spark.load_model(model_production_uri)

# Transition model stage
client.transition_model_version_stage(
    name="AirbnbPriceModel",
    version=2,
    stage="Production"
)
```

### Streaming Deployment

```python
# Load saved model
pipelineModel = mlflow.spark.load_model(f"runs:/{run_id}/model")

# Set up streaming data
repartitionedPath = "/path/to/streaming-data"
schema = spark.read.parquet(repartitionedPath).schema

streamingData = (spark
                 .readStream
                 .schema(schema)
                 .option("maxFilesPerTrigger", 1)
                 .parquet(repartitionedPath))

# Generate predictions
streamPred = pipelineModel.transform(streamingData)

# Write to sink
(streamPred.writeStream
 .format("parquet")
 .option("path", "/path/to/output")
 .option("checkpointLocation", "/path/to/checkpoint")
 .start())
```

### Real-time Inference with XGBoost

```scala
// Train XGBoost model
import ml.dmlc.xgboost4j.scala.spark.XGBoostRegressor

val xgboost = new XGBoostRegressor()
  .setFeaturesCol("features")
  .setLabelCol("price")
  .setMaxDepth(5)
  .setNumRound(100)

// Extract native model for serving
val xgboostModel = xgboostPipelineModel.stages.last
  .asInstanceOf[XGBoostRegressionModel]
xgboostModel.nativeBooster.saveModel("/path/to/xgboost_native_model")
```

```python
# Serve with XGBoost Python
import xgboost as xgb
bst = xgb.Booster({'nthread': 4})
bst.load_model("/path/to/xgboost_native_model")
```

### Pandas UDFs for Distributed Inference

```python
import mlflow.sklearn
import pandas as pd

def predict(iterator):
    # Load model once (not per batch)
    model_path = f"runs:/{run_id}/random-forest-model"
    model = mlflow.sklearn.load_model(model_path)
    
    for features in iterator:
        yield pd.DataFrame(model.predict(features))

# Apply in parallel
df.mapInPandas(predict, "prediction double").show(3)
```

### Distributed Hyperparameter Tuning with Hyperopt

```python
import hyperopt
from hyperopt import SparkTrials, fmin, tpe, hp

# Define search space
space = {
    'max_depth': hp.choice('max_depth', [2, 5, 10]),
    'num_trees': hp.choice('num_trees', [20, 50, 100])
}

# Define training function
def train_fn(params):
    # Train model with params
    # Return loss (e.g., RMSE)
    return loss

# Run distributed search
best_hyperparameters = fmin(
    fn=train_fn,
    space=space,
    algo=tpe.suggest,
    max_evals=64,
    trials=SparkTrials(parallelism=4)
)
```

---
