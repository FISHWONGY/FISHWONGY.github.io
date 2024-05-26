---
authors:
- Hugo Authors
date: "2024-05-18"
excerpt: Setting up GCP Databricks with a sample end-to-end ML project
hero: /images/on_gcpdatabricks/cover_img.png
title: On GCP Databricks
---

A while ago, I published a [post](https://fishwongy.github.io/post/20231023_azuredatabricks/) discussing how to set up a Databricks environment on Azure. 

This time, I will be discussing how to set up a Databricks environment on GCP, then we will try to extract some data from Open Weather API to Databricks, and finally, we will try to predict the temperature using GBT Regressor.

<u><b>
    <p style="font-size:20pt ">
      Setting up GCP Databricks
</b></u>

1 - GCP console authentication

To directly just GCP's console on the browser, we need to set up the authentication and download databricks-cli with brew. 

```bash
sudo passwd $USER

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# Enter password that we just configure

(echo; echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"') >> /home/$USER/.bashrc

eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"

brew tap databricks/tap
brew install databricks
```

Before we configure the databricks-cli, we need to obtain some required information.

If we navigate to the Databricks Admin UI, we can find the URL of the workspace. 
After we have the URL, and enter Databricks workspace, navigate to User Settings > Develop > Access Tokens > Generate New Token.


Going back to GCP console and execute
```bash
databricks configure
```

Follow the prompts and enter the required information. 

Account will be something like `https://123.x.gcp.databricks.com/` and token will be the token we generated from the Databricks workspace above.


2 - Create secret scope

Databricks handles sensitive information with secret scopes, a secure way to store API keys, passwords, and tokens.
Here, we will store our Open Weather API key in a secret scope and credential for our service account.

a) Create a new secret scope
```bash
databricks secrets create-scope my-db-scope
```

b) Create a new secret in the scope
```bash
databricks secrets put-secret --json '{
  "scope": "my-db-scope",
  "key": "own-api",
  "string_value": "your-key"
}'
```

c) To check the secret
```bash
databricks secrets list-secrets my-db-scope
```

d) Access the secret via Databricks notebook
```python
dbutils.secrets.get(scope="my-db-scope", key="own-api")
```

3 - Create and configure databricks cluster

I created just a personal cluster, but you can create a cluster depends on your needs.

In GCP, when we create a cluster, it will automatically Kubernetes cluster for us on our GCP project.

If we need to access data in our GCP storaage via databricks, we will have to add the below configurations in the cluster's advanced options.
```bash
spark.hadoop.google.cloud.auth.service.account.enable true
spark.hadoop.fs.gs.project.id 	gcp-prj-123
spark.hadoop.fs.gs.auth.service.account.private.key {{secrets/my-db-scope/private-key}}
spark.hadoop.fs.gs.auth.service.account.email {{secrets/my-db-scope/client-email}}
spark.hadoop.fs.gs.auth.service.account.private.key.id {{secrets/my-db-scope/private-key-id}}
```


<u><b>
    <p style="font-size:20pt ">
      Working with Open Weather API
</b></u>

First, let's start with creating a schema in Databricks.

```sql
CREATE SCHEMA BRONZE_SCHEMA;
```

Then let's define some functions for extracting data from Open Weather API to a databricks table, and transform the temperature from Kelvin to Celsius.
```python
import numpy as np
import requests
from datetime import datetime
import json
from pyspark.sql import DataFrame



API_KEY = dbutils.secrets.get(scope="my-db-scope", key="own-api1")
URL = 'https://api.openweathermap.org/data/2.5/weather'

def request_weather_data(location: str, api_key: str) -> dict:
    params = {'q': location, 'appid': api_key}
    response = requests.get(URL, params=params)
    data = response.json()
    
    data['main']['temp'] = np.round((data['main']['temp'] - 273.15), 2)
    data['main']['feels_like'] = np.round((data['main']['feels_like'] - 273.15), 2)
    data['main']['temp_min'] = np.round((data['main']['temp_min'] - 273.15), 2)
    data['main']['temp_max'] = np.round((data['main']['temp_max'] - 273.15), 2)
    
    data['timestamp'] = datetime.now().strftime("%m-%d-%y %H:%M:%S.%f")

    return data


def dict_to_df(data: dict) -> DataFrame:
    json_data = json.dumps(data)
    json_list = []
    json_list.append(json_data)
    jsonRDD = sc.parallelize(json_list)
    df = spark.read.json(jsonRDD)

    return df


def write_df_delta(dataframe: DataFrame, table_name: str):
    dataframe.write.format('delta') \
        .mode('append') \
        .option("mergeSchema", "true") \
        .saveAsTable(f'{table_name}')
```

And to write data to our datbricks table

```python
weather = request_weather_data('London,GB', API_KEY)
df = dict_to_df(weather)
write_df_delta(df, "BRONZE_SCHEMA.WEATHER_DATA")
```


Apart from writing data as batch, we can also write data as stream.

```python
import dlt
from pyspark.sql.functions import *
from pyspark.sql.types import *

KAFKA_SERVER = dbutils.secrets.get(scope = "my-db-scope", key = "kafka-server")
KAFKA_USER = dbutils.secrets.get(scope = "my-db-scope", key = "kafka-user")
KAFKA_PASSWORD = dbutils.secrets.get(scope = "my-db-scope", key = "kafka-pass")

# To coonect to a topic stream from RedPanda Kafka
raw_kafka_events = (spark.readStream.format("kafka") \
.option("kafka.bootstrap.servers", f"{KAFKA_SERVER}") \
.option("subscribe", "owm-events") \
.option("kafka.group.id", "owm-consumer") \
.option("kafka.security.protocol", "SASL_SSL") \
.option("kafka.sasl.mechanism", "SCRAM-SHA-256") \
.option("kafka.sasl.jaas.config", f"kafkashaded.org.apache.kafka.common.security.scram.ScramLoginModule required username=\"{KAFKA_USER}\" password=\"{KAFKA_PASSWORD}\";") \
.option("startingOffsets", "earliest") \
.option("failOnDataLoss", "false") \
.load()
)
```

```python
spark.sql("USE BRONZE_SCHEMA")

# Define a DLT table
@dlt.table(table_properties={"pipelines.reset.allowed":"false"})
def o365_events_raw():
  return raw_kafka_events
```


Finally, using the data we gathered from the above process, let's try to use GBTs learning algorithm to predict the temperature.

```python
import pyspark.sql.functions as F
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler, VectorIndexer
from pyspark.ml.regression import GBTRegressor
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder

from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml import Pipeline



df = spark.sql("SELECT * FROM BRONZE_SCHEMA.WEATHER_DATA")
df.limit(5).display()
```


Then we will transform the data to a more readable format and index the categorical columns and select the features we will use for the model.
```python
df2 = df.withColumn("coord_lat", F.col("coord.lat")) \
       .withColumn("coord_lon", F.col("coord.lon")) \
       .withColumn("temp_feels_like", F.col("main.feels_like")) \
       .withColumn("temp", F.col("main.temp")) \
       .withColumn("temp_max", F.col("main.temp_max")) \
       .withColumn("temp_min", F.col("main.temp_min")) \
       .withColumn("humidity", F.col("main.humidity")) \
       .withColumn("pressure", F.col("main.pressure")) \
       .withColumn("country", F.col("sys.country")) \
       .withColumn("wind_deg", F.col("wind.deg")) \
       .withColumn("wind_speed", F.col("wind.speed")) \
       .withColumn("weather_description", F.explode(F.col("weather.description"))) \
       .withColumn("weather_main", F.explode(F.col("weather.main")))\
       .withColumn("day", F.dayofmonth(F.to_timestamp(F.col("timestamp"), "MM-dd-yy HH:mm:ss.SSSSSS"))) \
       .withColumn("hour", F.hour(F.to_timestamp(F.col("timestamp"), "MM-dd-yy HH:mm:ss.SSSSSS"))) \
       .select("day", "hour", "country", "coord_lat", "coord_lon", "temp_feels_like", "temp", "temp_max", "temp_min",
               "humidity", "pressure", "wind_deg", "wind_speed",
               "weather_description", "weather_main")

df2.display()
df2.printSchema()
```

```python
indexers = StringIndexer(
    inputCols=['country', 'weather_description', 'weather_main'], 
    
    outputCols=['country_num', 'weather_description_num', 'weather_main_num']).setHandleInvalid("keep").fit(df2)

full_features_indexed_df = indexers.transform(df2)
full_features_indexed_df = full_features_indexed_df.drop('country').drop('weather_description').drop('weather_main')
```

Let's take a look at the data and schema
```python
full_features_indexed_df.display()
full_features_indexed_df.printSchema()
```

Split the dataset randomly into 70% for training and 30% for testing. Passing a seed for deterministic behavior
```python
train, test = full_features_indexed_df.randomSplit([0.7, 0.3], seed = 0)

print("There are %d training examples and %d test examples." % (train.count(), test.count()))
display(train.select("hour", "temp"))
```

## Train the machine learning pipeline

Now that we have reviewed the data and prepared it as a DataFrame with numeric values, we are ready to train a model to predict future bike sharing rentals.

Most MLlib algorithms require a single input column containing a vector of features and a single target column. The DataFrame currently has one column for each feature. MLlib provides functions to help us prepare the dataset in the required format.

MLlib pipelines combine multiple steps into a single workflow, making it easier to iterate as you develop the model.

In this example, we create a pipeline using the following functions:

- VectorAssembler: Assembles the feature columns into a feature vector.
- VectorIndexer: Identifies columns that should be treated as categorical.
- GBTRegressor: Uses the Gradient-Boosted Trees (GBT) algorithm to learn how to predict rental counts from the feature vectors.
- CrossValidator: The GBT algorithm has several hyperparameters. This notebook illustrates how to use hyperparameter tuning in Spark. This capability automatically tests a grid of hyperparameters and chooses the best resulting model.


For more information:
[VectorAssembler](https://spark.apache.org/docs/latest/ml-features.html#vectorassembler), 
[VectorIndexer](https://spark.apache.org/docs/latest/ml-features.html#vectorindexer)

Remove the target column from the input feature set.

```python
featuresCols = full_features_indexed_df.columns

featuresCols.remove('temp')
```

vectorAssembler combines all feature columns into a single feature vector column, "rawFeatures".

```python
vectorAssembler = VectorAssembler(inputCols=featuresCols, outputCol="rawFeatures")
```

vectorIndexer identifies categorical features and indexes them, and creates a new column "features". 

```python
vectorIndexer = VectorIndexer(inputCol="rawFeatures", outputCol="features", maxCategories=4)
```

Next, define the model.

The following command defines a GBTRegressor model that takes an input column "features" by default and learns to predict the labels in the "temp" column. 
```python
gbt = GBTRegressor(labelCol="temp")
```

The third step is to wrap the model you just defined in a CrossValidator stage. CrossValidator calls the GBT algorithm with different hyperparameter settings. It trains multiple models and selects the best one, based on minimizing a specified metric. In this example, the metric is root mean squared error (RMSE).

```python
# Define a grid of hyperparameters to test:
#  - maxDepth: maximum depth of each decision tree
#  - maxIter: iterations, or the total number of trees
paramGrid = (ParamGridBuilder()
  .addGrid(gbt.maxDepth, [2, 5])
  .addGrid(gbt.maxIter, [10, 100])
  .build())

# Define an evaluation metric. The CrossValidator compares the true labels with predicted values for each combination of parameters, and calculates this value to determine the best model.
evaluator = RegressionEvaluator(metricName="rmse", labelCol=gbt.getLabelCol(), predictionCol=gbt.getPredictionCol())

# Declare the CrossValidator, which performs the model tuning.
cv = CrossValidator(estimator=gbt, evaluator=evaluator, estimatorParamMaps=paramGrid)
```

Create the pipeline.
```python
pipeline = Pipeline(stages=[vectorAssembler, vectorIndexer, cv])
```

Train the pipeline.

When calling fit(), the pipeline runs feature processing, model tuning, and training and returns a fitted pipeline with the best model it found. This step takes several minutes.
```python
pipelineModel = pipeline.fit(train)
```

Make predictions and evaluate results.

The final step is to use the fitted model to make predictions on the test dataset and evaluate the model's performance. The model's performance on the test dataset provides an approximation of how it is likely to perform on new data.
Computing evaluation metrics is important for understanding the quality of predictions, as well as for comparing models and tuning parameters.

```python
predictions = pipelineModel.transform(test)

display(predictions.select("temp", "prediction", *featuresCols))
```

A common way to evaluate the performance of a regression model is the calculate the root mean squared error (RMSE). The value is not very informative on its own, but you can use it to compare different models. CrossValidator determines the best model by selecting the one that minimizes RMSE. 

```python
rmse = evaluator.evaluate(predictions)

print("RMSE on our test set: %g" % rmse)

```

It's also a good idea to examine the residuals, or the difference between the expected result and the predicted value. The residuals should be randomly distributed; if there are any patterns in the residuals, the model may not be capturing something important. In this case, the average residual is about 1, less than 1% of the average value of the `temp`column.
```python
predictions_with_residuals = predictions.withColumn("residual", (F.col("temp") - F.col("prediction")))

display(predictions_with_residuals.agg({'residual': 'mean'}))
```

As an optional final step, let's save the model and/ or pipeline for future use. A more intuitive way to do so is with MLFlow with its model serving capabilities, however its integration with GCP is still relativelty limited at this point compare to Azure and AWS. 

In our case, we will save the model to our GCS file system this time.

Save the pipeline model to GCS
```python
# Specify the GCS directory path
gcs_model_directory = "gs://ml-models-checkpoint/owm-weather/gbtregressor/pipelineModel"

# List the contents of the GCS directory
model_files = dbutils.fs.ls(gcs_model_directory)

# Filter and sort the files to find the most recent model's path
model_paths = sorted([file.path for file in model_files if file.isDir()], reverse=True)

# Get the most recent model's path
most_recent_model_path = model_paths[0]

model_ver = int(most_recent_model_path.split("_")[-1].rstrip("/")) + 1

# Save the model
pipelineModel.save(f'gs://ml-models-checkpoint/owm-weather/gbtregressor/pipelineModel/owm_pipelineModel_00{model_ver}')
```

Save the pipeline to GCS
```python
# Specify the GCS directory path
gcs_pipeline_directory = "gs://ml-models-checkpoint/owm-weather/gbtregressor/pipeline"


pipeline_files = dbutils.fs.ls(gcs_pipeline_directory)
pipeline_paths = sorted([file.path for file in pipeline_files if file.isDir()], reverse=True)
most_recent_pipeline_path = pipeline_paths[0]
pipeline_ver = int(most_recent_pipeline_path.split("_")[-1].rstrip("/")) + 1

# Save the pipeline that created the model
pipeline.save(f'gs://ml-models-checkpoint/owm-weather/gbtregressor/pipeline/owm_pipeline_00{pipeline_ver}')
```

To save the model and pipeline in a more reusable way, we can define a function as below.
```python
def save_ml_model(project_name: str, ml_model: str, model_name: str, model_object, model_type: str):
    if model_type not in ['pipeline', 'pipelineModel']:
        raise ValueError("model type must be 'pipeline' or 'pipelineModel'")
    
    gcs_model_directory = f"gs://ml-models-checkpoint/{project_name}/{ml_model}/{model_type}"
    
    try:
        model_files = dbutils.fs.ls(gcs_model_directory)
        model_paths = sorted([file.path for file in model_files if file.isDir()], reverse=True)
        most_recent_model_path = model_paths[0]
        model_ver = int(most_recent_model_path.split("_")[-1].rstrip("/")) + 1
    except IndexError:
        # If there are no existing models, start versioning from 1
        model_ver = 1
    
    # Save the ml object
    model_object.save(f'{gcs_model_directory}/{model_name}_{model_type}_00{model_ver}')
```

Then for example, we can save the model and pipeline as below.
```python
save_ml_model(
    project_name="owm-weather",
    ml_model="gbtregressor",
    model_name="owm",
    model_object=pipeline,  
    model_type='pipeline'  
)

save_ml_model(
    project_name="owm-weather",
    ml_model="gbtregressor",
    model_name="owm",
    model_object=pipelineModel,  
    model_type='pipelineModel'  
)
```

To use the model in the future, we can load the model and pipeline as below.
```python
# Specify the GCS directory path
gcs_model_directory = "gs://ml-models-checkpoint/owm-weather/gbtregressor/pipelineModel"

# List the contents of the GCS directory
model_files = dbutils.fs.ls(gcs_model_directory)

# Filter and sort the files to find the most recent model's path
model_paths = sorted([file.path for file in model_files if file.isDir()], reverse=True)

# Get the most recent model's path
most_recent_model_path = model_paths[0]
most_recent_model_path = most_recent_model_path.rstrip('/')
loaded_pipelineModel = PipelineModel.load(most_recent_model_path)
```

Use the loaded model, we just use it with the transform method as below.
```python
preds = loaded_pipelineModel.transform(df)
```

And that's it! We have successfully set up a Databricks environment on GCP, extracted data from Open Weather API to Databricks, predicted the temperature with Boosting and build a ML pipeline with that.

Thank you for reading and have a nice day!