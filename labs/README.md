## Lab: Writing an ETL pipeline using Apache Beam and Cloud Dataflow

Source Code: basic_etl.py

Build a batch Extract-Transform-Load pipeline in Apache Beam, which takes raw data from Google Cloud Storage and writes it to Google BigQuery.

The input data is intended to resemble web server logs in Common Log format along with other data that a web server might contain. The data is treated as a batch source. The task is to read the data, parse it, and then write it to BigQuery, a serverless data warehouse, for later data analysis. 

#### Create Virtual Environment 
```
sudo apt-get update && sudo apt-get install -y python3-venv 
python3 -m venv df-env 
source df-env/bin/activate 
```

#### Install Packages
```
python3 -m pip install -q --upgrade pip setuptools wheel 
python3 -m pip install apache-beam[gcp] 
```

#### Enable Dataflow API 
```
gcloud services enable dataflow.googleapis.com 
```

#### Generate Synthetic Data
```
cd ../scripts 
source create_batch_sinks.sh 
bash generate_batch_events.sh 
head events.json 
```

#### Run the pipelines using DirectRunner or Dataflow
```
export PROJECT_ID=$(gcloud config get-value project) 

python3 basic_etl.py \ 
  --project=${PROJECT_ID} \ 
  --region=us-central1 \ 
  --stagingLocation=gs://$PROJECT_ID/staging/ \ 
  --tempLocation=gs://$PROJECT_ID/temp/ \ 
  --runner=DirectRunner 
 
python3 basic_etl.py \ 
  --project=${PROJECT_ID} \ 
  --region=us-central1 \ 
  --stagingLocation=gs://$PROJECT_ID/staging/ \ 
  --tempLocation=gs://$PROJECT_ID/temp/ \ 
  --runner=DataflowRunner 
```

## Lab: Batch Analytics Pipelines with Cloud Dataflow

Part 1, For this lab, write a pipeline that: 
- Reads the day’s traffic from a file in Cloud Storage. 
- Converts each event into a CommonLog object. 
- Sums the number of hits for each unique user by grouping each object by user ID and combining the values to get the total number of hits for that particular user. 
- Performs additional aggregations on each user. 
- Writes the resulting data to BigQuery. 

Source Code: batch_user_traffic_pipeline.py

#### Setting up virtual environment and dependencies 
```
sudo apt-get update && sudo apt-get install -y python3-venv 
```

#### Create and activate virtual environment 
```
python3 -m venv df-env 
source df-env/bin/activate 
```

#### Install Packages 
```
python3 -m pip install -q --upgrade pip setuptools wheel 
python3 -m pip install apache-beam[gcp]
```
 
#### Enable Dataflow API 
```
gcloud services enable dataflow.googleapis.com 
```
 
#### Create GCS buckets and BQ dataset 
```
cd ../scripts
source create_batch_sinks.sh 
```

#### Generate event dataflow 
```
source generate_batch_events.sh 
```

#### Run the Pipeline
```
export PROJECT_ID=$(gcloud config get-value project) 
export REGION=us-west1 
export BUCKET=gs://${PROJECT_ID} 
export PIPELINE_FOLDER=${BUCKET} 
export RUNNER=DataflowRunner 
export INPUT_PATH=${PIPELINE_FOLDER}/events.json 
export TABLE_NAME=${PROJECT_ID}:logs.user_traffic 

python3 batch_user_traffic_pipeline.py \ 
--project=${PROJECT_ID} \ 
--region=${REGION} \ 
--staging_location=${PIPELINE_FOLDER}/staging \ 
--temp_location=${PIPELINE_FOLDER}/temp \ 
--runner=${RUNNER} \ 
--input_path=${INPUT_PATH} \ 
--table_name=${TABLE_NAME} 
```

Part 2, For this lab, create a pipeline to aggregate by when events occurred.

Source Code: batch_minute_traffic_pipeline.py

#### Run the pipeline 
```
export PROJECT_ID=$(gcloud config get-value project) 
export REGION=us-west1 
export BUCKET=gs://${PROJECT_ID} 
export PIPELINE_FOLDER=${BUCKET} 
export RUNNER=DataflowRunner 
export INPUT_PATH=${PIPELINE_FOLDER}/events.json 
export TABLE_NAME=${PROJECT_ID}:logs.minute_traffic 

python3 batch_minute_traffic_pipeline.py \ 
--project=${PROJECT_ID} \ 
--region=${REGION} \ 
--staging_location=${PIPELINE_FOLDER}/staging \ 
--temp_location=${PIPELINE_FOLDER}/temp \ 
--runner=${RUNNER} \ 
--input_path=${INPUT_PATH} \ 
--table_name=${TABLE_NAME} 
```

## Lab: Streaming Analytics Pipeline with Cloud Dataflow

This lab take many of the concepts introduced in a batch context and apply them in a streaming context. The finished pipeline will first read JSON messages from PubSub and parse those messages before branching. One branch writes some raw data to BigQuery and takes note of event and processing time. The other branch windows and aggregates the data and then writes the results to BigQuery. 
- Read data from a streaming source. 
- Write data to a streaming sink. 
- Window data in a streaming context. 
- Experimentally verify the effects of lag. 

Source Code: streaming_minute_traffic_pipeline.py

#### Setting up virtual environment and dependencies 
```
sudo apt-get install -y python3-venv 
```

#### Create and activate virtual environment 
```
python3 -m venv df-env 
source df-env/bin/activate 
```

#### Install Packages 
```
python3 -m pip install -q --upgrade pip setuptools wheel 
python3 -m pip install apache-beam[gcp] 
```

#### Enable the Dataflow API 
```
gcloud services enable dataflow.googleapis.com 
```

#### Grant the dataflow.worker role to the Compute Engine default service 
```
PROJECT_ID=$(gcloud config get-value project) 
export PROJECT_NUMBER=$(gcloud projects list --filter="$PROJECT_ID" --format="value(PROJECT_NUMBER)") 
export serviceAccount=""$PROJECT_NUMBER"-compute@developer.gserviceaccount.com" 
```
 
In the Cloud Console, navigate to IAM & ADMIN > IAM, click on Edit principal icon for Compute Engine default service account. Add Dataflow Worker as another role and click Save.

#### Set up the Data Environment 
```
cd .../scripts/
source create_streaming_sinks.sh 
```

#### Run the pipeline 

The pipeline will start and connect to the PubSub topic, awaiting input; there is none currently. 
```
export PROJECT_ID=$(gcloud config get-value project) 
export REGION='us-central1' 
export BUCKET=gs://${PROJECT_ID} 
export PIPELINE_FOLDER=${BUCKET} 
export RUNNER=DataflowRunner 
export PUBSUB_TOPIC=projects/${PROJECT_ID}/topics/my_topic 
export WINDOW_DURATION=60 
export AGGREGATE_TABLE_NAME=${PROJECT_ID}:logs.windowed_traffic 
export RAW_TABLE_NAME=${PROJECT_ID}:logs.raw 
python3 streaming_minute_traffic_pipeline.py \ 
--project=${PROJECT_ID} \ 
--region=${REGION} \ 
--staging_location=${PIPELINE_FOLDER}/staging \ 
--temp_location=${PIPELINE_FOLDER}/temp \ 
--runner=${RUNNER} \ 
--input_topic=${PUBSUB_TOPIC} \ 
--window_duration=${WINDOW_DURATION} \ 
--agg_table_name=${AGGREGATE_TABLE_NAME} \ 
--raw_table_name=${RAW_TABLE_NAME} 
```

#### Generate lag-less streaming input 
```
bash generate_streaming_events.sh 
```
 
#### Generate lag streaming input 
```
bash generate_streaming_events.sh true 
```

## Lab: Advanaced Streaming Analytics Pipeline with Cloud Dataflow

This lab introduces Apache Beam concepts that allow pipeline creators to specify how their pipelines should deal with lag in a formal way.
  - Deal with late data
  - Deal with malformed data by:
    - Writing a composite transform for more modular code
    - Writing a transform that emits multiple outputs of different types
    - Collecting malformed data and writing it to a location where it can be examined

Source Code: advanced_streaming_minute_traffic_pipeline.py

#### Setting up virtual environment and dependencies 
```
sudo apt-get update && sudo apt-get install -y python3-venv 
```

#### Create and activate virtual environment 
```
python3 -m venv df-env 
source df-env/bin/activate 
```

#### Install Packages 
```
python3 -m pip install -q --upgrade pip setuptools wheel 
python3 -m pip install apache-beam[gcp] 
```

#### Enable the Dataflow API 
```
gcloud services enable dataflow.googleapis.com 
```

#### Set up the Data Environment 
```
cd .../scripts/
source create_streaming_sinks.sh 
```

#### Run the pipeline 

The pipeline will start and connect to the PubSub topic, awaiting input; there is none currently. 
```
export PROJECT_ID=$(gcloud config get-value project) 
export REGION=us-central1 
export BUCKET=gs://${PROJECT_ID} 
export PIPELINE_FOLDER=${BUCKET} 
export RUNNER=DataflowRunner 
export PUBSUB_TOPIC=projects/${PROJECT_ID}/topics/my_topic 
export WINDOW_DURATION=60 
export ALLOWED_LATENESS=1 
export OUTPUT_TABLE_NAME=${PROJECT_ID}:logs.minute_traffic 
export DEADLETTER_BUCKET=${BUCKET} 

python3 streaming_minute_traffic_pipeline.py \ 
--project=${PROJECT_ID} \ 
--region=${REGION} \ 
--staging_location=${PIPELINE_FOLDER}/staging \ 
--temp_location=${PIPELINE_FOLDER}/temp \ 
--runner=${RUNNER} \ 
--input_topic=${PUBSUB_TOPIC} \ 
--window_duration=${WINDOW_DURATION} \ 
--allowed_lateness=${ALLOWED_LATENESS} \ 
--table_name=${OUTPUT_TABLE_NAME} \ 
--dead_letter_bucket=${DEADLETTER_BUCKET} \ 
--allow_unsafe_triggers 
```
 
#### Generate streaming input 
```
bash generate_streaming_events.sh true 
```

## Reference to Lab Content

git clone https://github.com/GoogleCloudPlatform/training-data-analyst 

cd /home/jupyter/training-data-analyst/quests/dataflow_python/ 
