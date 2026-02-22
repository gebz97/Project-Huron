
## Chapter 10: Machine Learning with MLlib

### Machine Learning Types

**Supervised Learning:**
- **Classification** - Predict discrete labels (dog/cat/spam)
- **Regression** - Predict continuous values (price/temperature)

**Unsupervised Learning:**
- **Clustering** - Find natural groupings
- **Dimensionality Reduction** - Reduce number of features

### MLlib Terminology

| Term | Description | Method |
|------|-------------|--------|
| **Transformer** | Converts DataFrame → DataFrame | `.transform()` |
| **Estimator** | Learns from DataFrame → Model | `.fit()` |
| **Pipeline** | Chains transformers/estimators | `.fit()` → PipelineModel |
| **Model** | Trained transformer | `.transform()` |

### Data Preparation

**Load Airbnb data:**
```python
filePath = "/databricks-datasets/learning-spark-v2/sf-airbnb/sf-airbnb-clean.parquet/"
airbnbDF = spark.read.parquet(filePath)

airbnbDF.select("neighbourhood_cleansed", "room_type", 
                "bedrooms", "price").show(5)
```

**Train/Test Split:**
```python
trainDF, testDF = airbnbDF.randomSplit([.8, .2], seed=42)
print(f"Train: {trainDF.count()}, Test: {testDF.count()}")
```

### VectorAssembler

```python
from pyspark.ml.feature import VectorAssembler

# Combine features into single vector
vecAssembler = VectorAssembler(inputCols=["bedrooms"], 
                                outputCol="features")
vecTrainDF = vecAssembler.transform(trainDF)
vecTrainDF.select("bedrooms", "features", "price").show(5)
```

### Linear Regression

```python
from pyspark.ml.regression import LinearRegression

# Create estimator
lr = LinearRegression(featuresCol="features", labelCol="price")

# Fit model
lrModel = lr.fit(vecTrainDF)

# View coefficients
m = round(lrModel.coefficients[0], 2)
b = round(lrModel.intercept, 2)
print(f"price = {m}*bedrooms + {b}")
```

### Creating a Pipeline

```python
from pyspark.ml import Pipeline

# Create pipeline
pipeline = Pipeline(stages=[vecAssembler, lr])

# Fit pipeline
pipelineModel = pipeline.fit(trainDF)

# Transform test data
predDF = pipelineModel.transform(testDF)
predDF.select("bedrooms", "features", "price", "prediction").show(10)
```

### One-Hot Encoding

```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder

# Identify categorical columns
categoricalCols = [field for (field, dataType) in trainDF.dtypes 
                   if dataType == "string"]

# StringIndexer
indexOutputCols = [x + "Index" for x in categoricalCols]
stringIndexer = StringIndexer(inputCols=categoricalCols, 
                              outputCols=indexOutputCols, 
                              handleInvalid="skip")

# OneHotEncoder
oheOutputCols = [x + "OHE" for x in categoricalCols]
oheEncoder = OneHotEncoder(inputCols=indexOutputCols, 
                           outputCols=oheOutputCols)

# Numeric columns
numericCols = [field for (field, dataType) in trainDF.dtypes 
               if ((dataType == "double") & (field != "price"))]

# Combine all features
assemblerInputs = oheOutputCols + numericCols
vecAssembler = VectorAssembler(inputCols=assemblerInputs, 
                                outputCol="features")
```

### RFormula (Simpler Alternative)

```python
from pyspark.ml.feature import RFormula

# RFormula handles StringIndexer + OneHotEncoder automatically
rFormula = RFormula(formula="price ~ .", 
                     featuresCol="features", 
                     labelCol="price", 
                     handleInvalid="skip")
```

### Pipeline with All Features

```python
# Using explicit stages
pipeline = Pipeline(stages=[stringIndexer, oheEncoder, vecAssembler, lr])

# Or using RFormula
pipeline = Pipeline(stages=[rFormula, lr])

pipelineModel = pipeline.fit(trainDF)
predDF = pipelineModel.transform(testDF)
```

### Model Evaluation

**RMSE (Root Mean Square Error):**
```python
from pyspark.ml.evaluation import RegressionEvaluator

regressionEvaluator = RegressionEvaluator(
    predictionCol="prediction", 
    labelCol="price", 
    metricName="rmse")

rmse = regressionEvaluator.evaluate(predDF)
print(f"RMSE: {rmse:.1f}")
```

**R² (R-squared):**
```python
r2 = regressionEvaluator.setMetricName("r2").evaluate(predDF)
print(f"R²: {r2}")
```

### Saving and Loading Models

```python
# Save model
pipelinePath = "/tmp/lr-pipeline-model"
pipelineModel.write().overwrite().save(pipelinePath)

# Load model
from pyspark.ml import PipelineModel
savedPipelineModel = PipelineModel.load(pipelinePath)
```

### Decision Trees

```python
from pyspark.ml.regression import DecisionTreeRegressor

dt = DecisionTreeRegressor(labelCol="price", maxBins=40)

# Pipeline
stages = [stringIndexer, vecAssembler, dt]
pipeline = Pipeline(stages=stages)
pipelineModel = pipeline.fit(trainDF)

# View tree rules
dtModel = pipelineModel.stages[-1]
print(dtModel.toDebugString)

# Feature importance
featureImp = pd.DataFrame(
    list(zip(vecAssembler.getInputCols(), dtModel.featureImportances)),
    columns=["feature", "importance"])
featureImp.sort_values(by="importance", ascending=False)
```

### Random Forests

```python
from pyspark.ml.regression import RandomForestRegressor

rf = RandomForestRegressor(labelCol="price", maxBins=40, seed=42, 
                           maxDepth=5, numTrees=100)

pipeline = Pipeline(stages=[stringIndexer, vecAssembler, rf])
pipelineModel = pipeline.fit(trainDF)
```

### Hyperparameter Tuning with Cross-Validation

```python
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator

# Build parameter grid
paramGrid = (ParamGridBuilder()
             .addGrid(rf.maxDepth, [2, 4, 6])
             .addGrid(rf.numTrees, [10, 100])
             .build())

# Evaluator
evaluator = RegressionEvaluator(labelCol="price", 
                                 predictionCol="prediction", 
                                 metricName="rmse")

# CrossValidator
cv = CrossValidator(estimator=pipeline,
                    evaluator=evaluator,
                    estimatorParamMaps=paramGrid,
                    numFolds=3,
                    seed=42,
                    parallelism=4)  # Parallel training

# Fit
cvModel = cv.fit(trainDF)

# View results
list(zip(cvModel.getEstimatorParamMaps(), cvModel.avgMetrics))
```

### Optimizing Pipelines

**Put CrossValidator inside Pipeline:**
```python
# Instead of pipeline inside CV, put CV inside pipeline
cv = CrossValidator(estimator=rf,  # Not pipeline
                    evaluator=evaluator,
                    estimatorParamMaps=paramGrid,
                    numFolds=3,
                    parallelism=4,
                    seed=42)

pipeline = Pipeline(stages=[stringIndexer, vecAssembler, cv])
pipelineModel = pipeline.fit(trainDF)  # Faster!
```

---
