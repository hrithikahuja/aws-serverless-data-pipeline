# AWS Serverless Data Pipeline (4 Stages + Evidence)
 
It implements an end to end serverless data pipeline using AWS managed services, with code and evidence for each stage.

Core services: S3, Athena, Lambda, Step Functions, SNS  
Languages: SQL (Athena), Python (boto3)

## What This Pipeline Does

- Reads raw healthcare facility data stored in S3 as NDJSON (one JSON object per line)
- Creates curated analytics outputs in Parquet using Athena CTAS
- Filters facilities with accreditations expiring within a configurable time window using Python and boto3
- Runs an asynchronous Athena aggregation via Lambda
- Orchestrates polling, retries, and production publishing via Step Functions
- Sends failure notifications via SNS for observability

## High Level Architecture

1. **S3 Raw**: NDJSON files land in `raw/clean_dataset/`
2. **Athena External Table**: Athena queries raw NDJSON directly from S3
3. **Stage 1 (Athena CTAS)**: Write curated facility metrics to Parquet in `processed/stage1_facility_result/`
4. **Stage 2 (Python boto3)**: Filter expiring accreditations and write NDJSON to `results/expiring_facilities/`
5. **Stage 3 (Lambda)**: Submit Athena aggregation and return QueryExecutionId + expected output key
6. **Stage 4 (Step Functions)**: Wait + poll Athena, copy result to `production/state_counts/`, notify failures via SNS

## Repo Contents

- `stage1_athena_sql/`  
  Athena DDL and CTAS SQL queries for database, external table, and Stage 1 outputs

- `stage2_python/`  
  Python boto3 script for filtering expiring accreditations (CloudShell friendly)

- `stage3_lambda/`  
  Lambda function code that starts Athena aggregation and returns output metadata

- `stage4_step_functions/`  
  Step Functions state machine definition JSON (wait/poll/retry/copy/SNS)

- `evidence/`  
  Screenshots and logs proving execution for each stage:
  - Athena query runs and outputs
  - S3 folder structure and created artifacts
  - CloudWatch logs (Lambda execution, QueryExecutionId, expected output path)
  - Step Functions executions (SUCCEEDED / FAILED paths)
  - SNS failure alert email screenshots (failure test)

## Stage by Stage Summary (with outputs)

### Stage 1: Athena Setup + Curated Parquet Output
**Goal:** Make Athena read NDJSON from S3 and materialize a curated Parquet dataset.  
**Steps:**
- Create Athena database
- Create external table over NDJSON using JSON SerDe
- Run CTAS to create curated Parquet outputs in a dedicated S3 prefix
**Output:** Parquet dataset in `processed/stage1_facility_result/`  

### Stage 2: Python Processing (boto3)
**Goal:** Filter facilities where any accreditation expires within a time window (default 6 months).  
**Steps:**
- Read NDJSON records from S3
- Parse `valid_until` dates
- Filter matching facilities
- Write filtered NDJSON back to S3
**Output:** `expiring_facilities_<timestamp>.ndjson` in `results/expiring_facilities/`  

### Stage 3: Lambda Submits Athena Aggregation
**Goal:** Automate Athena query execution on new data events.  
**Steps:**
- Create Lambda IAM role (Athena, S3, Glue Catalog, CloudWatch logs)
- Create Lambda function with env vars for Athena DB and output location
- Trigger Lambda via S3 upload or Step Functions invocation
**Output:** QueryExecutionId and expected Athena output path in logs  

### Stage 4: Step Functions Orchestration + SNS Alerts
**Goal:** Make Athena async execution reliable using orchestration.  
**Steps:**
- Invoke Lambda to submit query
- Wait and poll Athena status via `getQueryExecution`
- On success, copy Athena CSV to production prefix
- On failure, publish SNS alert with full execution context
**Output:** Production CSV in `production/state_counts/` or SNS failure alert  

## Key Design Choices

- **NDJSON for Athena compatibility:** one JSON object per line prevents null parsing issues
- **CTAS to Parquet:** reduces Athena scan cost and improves query performance
- **No polling in Lambda:** Lambda submits Athena query only; Step Functions handles polling and retries
- **Dedicated S3 prefixes by stage/format:** avoids HIVE_BAD_DATA errors caused by mixed formats
- **SNS failure notifications:** improves observability and speeds up debugging

## How To Run (High Level)

1. Upload NDJSON into `s3://<bucket>/raw/clean_dataset/`
2. Run Stage 1 SQL in Athena to create external table and Parquet outputs
3. Run Stage 2 Python script in CloudShell (or locally with AWS credentials)
4. Deploy Stage 3 Lambda with required env vars and IAM permissions
5. Deploy Stage 4 Step Functions state machine and run an execution
6. Verify outputs in S3 and evidence in CloudWatch / Step Functions / SNS

## Notes

- Re running CTAS into the same output prefix may create duplicate files. Use a fresh output prefix or clean the folder before reruns.
- Athena output is typically written as `<QueryExecutionId>.csv` under the configured Athena output location.
