# End-to-End Batch Processing Pipeline with Spark on GCP

This project demonstrates how to build an end-to-end batch processing pipeline using Apache Spark on Google Cloud Platform (GCP). The pipeline processes source data files (`orders.csv` and `order_items.csv`), ingests them into MySQL hosted on Cloud SQL, and performs ETL operations using Dataproc.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup](#setup)
  - [1. Ingest Source Data to Cloud SQL](#1-ingest-source-data-to-cloud-sql)
  - [2. Create a Dataproc Cluster](#2-create-a-dataproc-cluster)
  - [3. Submit Dataproc Jobs](#3-submit-dataproc-jobs)
  - [4. Automate Jobs with Cloud Scheduler](#4-automate-jobs-with-cloud-scheduler)
- [Scripts](#scripts)
- [Usage](#usage)
- [License](#license)

---

## Overview
This pipeline is designed for batch processing of transactional data stored in CSV files. It uses GCP services like Cloud SQL, Dataproc, and Google Cloud Storage (GCS) to achieve the following:

1. Load source data into MySQL (Cloud SQL).
2. Extract data from MySQL and store it in GCS.
3. Perform ETL (Extract, Transform, Load) tasks using PySpark.

## Prerequisites
- Google Cloud account
- GCP SDK installed
- `gcloud` CLI configured
- Python 3.x installed locally

## Architecture
1. Source data (`orders.csv`, `order_items.csv`) is uploaded to Cloud SQL.
2. Dataproc is used to extract data from Cloud SQL to GCS.
3. An ETL job transforms the data and writes the output to a GCS bucket.

## Setup

### 1. Ingest Source Data to Cloud SQL
1. Create a Cloud SQL instance with MySQL:
   ```bash
   gcloud sql instances create mysql-instance \
       --tier=db-n1-standard-1 \
       --region=us-central1
   ```

2. Import the source data:
   - Upload `orders.csv` and `order_items.csv` to your Cloud Storage bucket.
   - Use the MySQL CLI or tools like `mysqlimport` to load the data into the Cloud SQL instance.

### 2. Create a Dataproc Cluster
Run the following command to create a Dataproc cluster:
```bash
gcloud dataproc clusters create mysql-etl \
    --image-version=2.1-ubuntu20 \
    --region=us-central1 \
    --enable-component-gateway \
    --num-masters=1 \
    --master-machine-type=n2-standard-2 \
    --worker-machine-type=n2-standard-2 \
    --master-boot-disk-size=30GB \
    --worker-boot-disk-size=30GB \
    --num-workers=2 \
    --initialization-actions=gs://pyspark-fs-sid/install.sh \
    --metadata 'PIP_PACKAGES=mysql-connector-python' \
    --optional-components=JUPYTER
```

### 3. Submit Dataproc Jobs

#### Part-1: Data Extraction Job (MySQL to GCS)
Submit the extraction job:
```bash
gcloud dataproc jobs submit pyspark pyspark-mysql-extraction.py \
    --cluster=mysql-etl \
    --region=us-central1
```

#### Part-2: ETL Job
Submit the ETL job:
```bash
gcloud dataproc jobs submit pyspark pyspark-etl-pipeline.py \
    --cluster=mysql-etl \
    --region=us-central1
```

### 4. Automate Jobs with Cloud Scheduler
Use Cloud Scheduler to automate the workflow execution:

1. Enable the Cloud Scheduler API:
   ```bash
   gcloud services enable cloudscheduler.googleapis.com
   ```

2. Add the required IAM policy binding:
   ```bash
   gcloud projects add-iam-policy-binding {project-id} \
       --member serviceAccount:{project-number}-compute@developer.gserviceaccount.com \
       --role roles/workflows.invoker
   ```

3. Create a Cloud Scheduler job:
   ```bash
   gcloud scheduler jobs create http spark-workflow-scheduler \
   --schedule="0 9 * * 1" \
   --uri="https://dataproc.googleapis.com/v1/projects/{project-id}/regions/us-central1/workflowTemplates/{wf-template-name}:instantiate?alt=json" \
   --location="us-central1" \
   --message-body="{\"argument\": \"trigger_workflow\"}" \
   --time-zone="America/New_York" \
   --oauth-service-account-email="{project-number}-compute@developer.gserviceaccount.com"
   ```

## Scripts
- `pyspark-mysql-extraction.py`: Extracts data from MySQL in Cloud SQL and uploads it to GCS.
- `pyspark-etl-pipeline.py`: Reads data from GCS, performs transformations, and saves the output back to GCS.

## Usage
1. Ensure the source CSV files are uploaded to Cloud Storage.
2. Load the data into MySQL in Cloud SQL.
3. Create a Dataproc cluster using the provided `gcloud` command.
4. Run the extraction and ETL jobs sequentially.
5. Check the output in your specified GCS bucket.
6. Use Cloud Scheduler to automate the pipeline.

## License
This project is licensed under the MIT License. See the LICENSE file for details.

