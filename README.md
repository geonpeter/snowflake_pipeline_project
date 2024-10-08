﻿# snowflake_pipeline_project
## Overview
This project sets up a pipeline for ingesting customer data into Snowflake from a Google Cloud Storage (GCS) bucket. It also demonstrates using Snowflake Snowpipe for continuous data ingestion and automates the process through scheduled tasks. Data is ingested into the main table and then filtered into a secondary table for UK-based customers.
## Workflow Steps:
- Set up database, schema, and tables in Snowflake.
-  Configure GCS storage integration to allow Snowflake to access files.
-  Set up a Snowpipe for continuous data ingestion.
-  Schedule tasks to periodically ingest and process the data.
- Monitor and manage tasks and Snowpipe.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Workflow Diagram
![Workflow Diagram](https://github.com/geonpeter/snowflake_pipeline_project/blob/master/workflow_customer_pjt_snowflake.png?raw=true)

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Steps
## 1. Create and Configure Snowflake Environment
```sql
-- Drop the database if it exists
DROP DATABASE customers_db;

-- Create a new database
CREATE OR REPLACE DATABASE customers_db;

-- Create a new schema
CREATE SCHEMA customers_sch;

-- Set the default database and schema
USE customers_db;
USE SCHEMA customers_sch;

-- Create the customers_data table
CREATE OR REPLACE TABLE customers_data (
    customer_pk number(38,0),
    salutation varchar(10),
    first_name varchar(20),
    last_name varchar(30),
    gender varchar(1),
    marital_status varchar(1),
    day_of_birth date,
    birth_country varchar(60),
    email_address varchar(50),
    city_name varchar(60),
    zip_code varchar(10),
    country_name varchar(20),
    gmt_timezone_offset number(10,2),
    preferred_cust_flag boolean,
    registration_time timestamp_ltz(9)
);
```

## 2. Create GCS Storage Integration
```sql
-- Create a storage integration with GCS
CREATE OR REPLACE STORAGE INTEGRATION integration_gcp_customers
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = GCS
    ENABLED = TRUE
    STORAGE_ALLOWED_LOCATIONS = ('gcs://bucketforcustomers/');

-- Describe the integration to get service account details
DESC INTEGRATION integration_gcp_customers;

-- Principal Value: kiaf00000@gcpeuropewest3-1-d3ca.iam.gserviceaccount.com
```
## 3. Create Stage and List Files
```sql
-- Create a stage in Snowflake pointing to the GCS bucket
CREATE OR REPLACE STAGE snowflake_stage_customers
    URL = 'gcs://bucketforcustomers/'
    STORAGE_INTEGRATION = integration_gcp_customers;

-- List available stages
SHOW STAGES;

-- List files in the GCS bucket stage
LIST @snowflake_stage_customers;
```
## 4. Set Up Notification Integration with GCP Pub/Sub
```sql
-- Create a notification integration for Pub/Sub
CREATE OR REPLACE NOTIFICATION INTEGRATION notification_pubsub
    TYPE = QUEUE
    NOTIFICATION_PROVIDER = GCP_PUBSUB
    ENABLED = TRUE
    GCP_PUBSUB_SUBSCRIPTION_NAME = 'projects/projectbigdatageon/subscriptions/topic_customers-sub';

-- Describe the notification integration
DESC INTEGRATION notification_pubsub;

-- Principal Value: kjaf00000@gcpeuropewest3-1-d3ca.iam.gserviceaccount.com
```
## 5. Create a Snowpipe for Data Ingestion
```sql
-- Create a Snowpipe to ingest data into customers_data
CREATE OR REPLACE PIPE snowflake_pipe
    AUTO_INGEST = TRUE
    INTEGRATION = notification_pubsub
    AS
    COPY INTO customers_data
    FROM @snowflake_stage_customers
    FILE_FORMAT = (TYPE='CSV')
    ON_ERROR = 'SKIP_FILE';

-- Show all pipes
SHOW PIPES;

-- Check the status of the pipe
SELECT SYSTEM$PIPE_STATUS('snowflake_pipe');

-- Check ingestion history for the last hour
SELECT * 
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(TABLE_NAME => 'customers_data', START_TIME => DATEADD(hours, -1, CURRENT_TIMESTAMP())));
```
## 6. Query Data
```sql
-- Query the customers_data table
SELECT * FROM customers_data;
```
## 7. Pause and Resume Snowpipe
```sql
-- Pause the Snowpipe
ALTER PIPE snowflake_pipe SET PIPE_EXECUTION_PAUSED = TRUE;

-- Resume the Snowpipe
ALTER PIPE snowflake_pipe SET PIPE_EXECUTION_PAUSED = FALSE;
```
### 8. Create UK Customer Table and Schedule Task
```sql
-- Create the customers_data_UK table
CREATE OR REPLACE TABLE customers_data_UK (
    customer_pk number(38,0),
    salutation varchar(10),
    first_name varchar(20),
    last_name varchar(30),
    gender varchar(1),
    marital_status varchar(1),
    day_of_birth date,
    birth_country varchar(60),
    email_address varchar(50),
    city_name varchar(60),
    zip_code varchar(10),
    country_name varchar(20),
    gmt_timezone_offset number(10,2),
    preferred_cust_flag boolean,
    registration_time timestamp_ltz(9)
);

-- Query the UK customers table
SELECT * FROM customers_data_UK;

-- Schedule a task to ingest data into customers_data_UK
CREATE OR REPLACE TASK target_table_ingestion
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON */2 * * * * UTC'  -- Every 2 minutes
    COMMENT = 'This task runs every 2 minutes'
AS
    INSERT INTO customers_data_UK SELECT * FROM customers_data WHERE country_name = 'UK';

-- Resume the task
ALTER TASK target_table_ingestion RESUME;

-- Check task history
SELECT * 
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(TASK_NAME => 'target_table_ingestion'))
ORDER BY SCHEDULED_TIME;

-- Query the customers_data_UK table
SELECT * FROM customers_data_UK;
```
## 9. Suspend and Drop the Task
```sql
-- Suspend the task
ALTER TASK target_table_ingestion SUSPEND;

-- Drop the task
DROP TASK target_table_ingestion;
```
## 10. Schedule a Maintenance Task
```sql
-- Schedule a task to delete old data from customers_data_UK
CREATE OR REPLACE TASK next_task
    WAREHOUSE = COMPUTE_WH
    AFTER target_table_ingestion
AS
    DELETE FROM customers_data_UK WHERE registration_time < CURRENT_DATE();
```
## 11. Monitoring and Cleanup
```sql
-- Check task history
SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(TASK_NAME => 'next_task')) ORDER BY SCHEDULED_TIME;

-- Drop the task when no longer needed
DROP TASK next_task;
```

------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Conclusion
This project demonstrates how to set up a Snowflake data pipeline using Snowpipe, GCS integration, and scheduled tasks to continuously ingest, process, and manage customer data. It also shows how to automate operations through task scheduling and manage pipeline execution states.
