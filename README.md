# AWS Glue ETL Pipeline: JSON to CSV Transformation
This project demonstrates a serverless ETL.

## Services Used:
* Amazon S3 (Simple Storage Service): Used for storing both the raw JSON input data and the transformed GZIP-compressed CSV output data.
* AWS Glue Crawler: Automatically scans data in S3, infers schemas, and populates the AWS Glue Data Catalog. In this demo, it also automatically creates the Glue database.
* AWS Glue Data Catalog: A centralized metadata repository that stores table definitions, schemas, and data locations. It acts as the "metadata store" for your data lake.
* AWS Glue Job (Visual ETL): A serverless Apache Spark-based ETL job configured using AWS Glue Studio's visual interface. It reads data from the Data Catalog, applies transformations, and writes the results to S3.
* AWS IAM (Identity and Access Management): Manages secure access and permissions for Glue components to interact with S3 and other services.

## Features
* Automated Schema Discovery: Glue Crawlers automatically detect data format and infer schema for quick data cataloging.
* Serverless ETL: No servers to provision or manage; AWS Glue handles all underlying infrastructure.
* Visual Job Authoring: AWS Glue Studio's visual editor simplifies the creation of complex ETL workflows.
* Data Transformation: Enables schema manipulation (e.g., dropping unwanted columns, casting data types) to prepare data for analytics.
* Format Conversion: Efficiently converts JSON data to compressed CSV, optimizing for storage and query performance.
* Scalable and Cost-Effective: Scales compute resources automatically based on data volume and complexity, with a pay-per-use model.

## Setup Guide
Follow these steps to deploy and run this JSON to CSV ETL pipeline in your AWS account.

* Prerequisites
An active AWS Account.

A JSON data file to use as input  converted to JSON Lines format by removing outer [] if it's a single array.

Step-by-Step Deployment
***************************************************************
# 1. Data Preparation and S3 Bucket Setup

Prepare your JSON Data:
Obtain your JSON data file. If it's a single JSON array ensure you convert it to JSON Lines format  by removing the outer [] brackets and ensuring each object is on a new line. 
Create S3 Buckets:
Navigate to the S3 Console.
Create three new S3 buckets in your preferred AWS Region:
Input Data Bucket: your-name-glue-input-data 
Output Data Bucket: your-name-glue-output-data 
Glue Scripts Bucket: your-name-glue-scripts 
Ensure "Block all public access" is enabled for all buckets for security.
Upload Input JSON Data:
Upload your prepared JSON data file to the your-name-glue-input-data-covid bucket.

# 2. Create an AWS IAM Role for Glue
This role grants Glue services the necessary permissions to read from S3, write to S3, and interact with the Glue Data Catalog.

Navigate to the IAM Console.
In the navigation pane, choose "Roles", then click "Create role".
Trusted Entity: Select "AWS service".
Use Case: Choose "Glue".
Click "Next".
Add Permissions: Attach the following policies:
AWSGlueServiceRole (usually auto-selected)
AmazonS3FullAccess (For demo simplicity; in production, scope down to specific buckets)
CloudWatchFullAccess (For job logging; scope down in production)
Click "Next".
Role Name: Enter name for the role
Click **"Create role"`.

# 3. Create AWS Glue Crawler (Crawler-First Approach)
This crawler will automatically create your Glue database and table from the JSON data.
Navigate to the AWS Glue Console.
In the navigation pane, under "Data Catalog", choose "Crawlers".
Click "Create crawler".
Crawler name: name_data_crawler_auto_db.
Click "Next".
Data sources:
Click "Add a data source".
Data source type: S3
S3 path: Browse to your s3://your-name-bucket/ bucket (where your JSON file is). Click "Add S3 data source".
"Is your data already mapped to Glue tables?": Select "Not yet".
Click "Next".
IAM role: Choose the created Role.
Click "Next".

Output configuration:
Target database: Click "Add database".
Database name: Enter name_db. Click "Create".
Ensure name_db is selected.

Click "Next".
Crawler schedule (Frequency): Select "On demand".
Click "Next".
Review and create: Click "Create crawler".
Run the crawler: Select name_data_crawler_auto_db and click **"Run"`. Wait for it to complete.
Verify Table Creation: Go to "Tables" under "Data Catalog". You should now see a table within the name_auto_db database, with a schema inferred from your JSON.

# 4. Create the AWS Glue Job (Visual ETL)
This job will read from your cataloged JSON data, transform it, and write it as GZIP-compressed CSV.
Navigate to the AWS Glue Console.
In the left-hand navigation pane, under "ETL", click on "Visual ETL".
Create Job: Click "Create". (This will open the visual editor).
Configure Job Properties (Job Details tab on right panel or first step of wizard):
Name: name_data_etl_job_auto
IAM role: select the Role created 
Glue version: Glue 4.0 (or latest recommended)
Worker type: G.1X
Automatically scale the number of workers: Check this box.
Script path: Browse to your s3://your-name-glue-bucket/ bucket.
Temporary path: Browse to your s3://your-name-glue-bucket/ bucket 
Job metrics: Check this box.
Continuous logging: Check this box.
Spark UI: Check "Standard - default".
Click "Save" (if this is a separate properties tab) or "Next" to proceed to the canvas.
Design the Visual ETL Workflow:


a. Configure the Data Source Node:
------------------------------------
On the canvas, click the "Data Source" node.
In the right panel, under "Node properties":
Node name: Givename_Data.
Data source: AWS Glue Data Catalog.
Under "Data source properties - Data Catalog":
Database: Select name_auto_db.
Table: Select the table created by your crawler (e.g., your_json_file_name).

b. Add a Transform Node (Select Fields/Drop Columns):
-----------------------------------------------------
Click the "+" icon on the canvas, or drag "Select fields" from the "Transforms" panel.
Connect the output of source node to the input of this new transform.
Click the "Select fields" node.
In the right panel, under "Node properties":
Node name: Drop_Unwanted_Columns.
Under the "Transform" tab:
Uncheck any columns you want to exclude from the output. Keep all necessary columns checked.

c. Add a Transform Node (Change Schema/Cast Types - Crucial for CSV Output):
--------------------------------------------------------------------------
Click the "+" icon on the canvas, or drag "Change Schema" from the "Transforms" panel.
Connect the output of Drop_Unwanted_Columns to the input of this new transform.
Click the "Change Schema" node.
In the right panel, under "Node properties":
Node name: Cast_Numeric_Columns.
Under the "Transform" tab:
Identify columns like new_cases and new_deaths (or any other numeric columns that might have been inferred as string).
For these columns, change their "Data type" from string to long (for whole numbers) or double (for decimals). 

d. Add a Data Target Node (S3 Output):
--------------------------------------------
Click the "+" icon on the canvas, or drag "Amazon S3" from the "Targets" panel.
Connect the output of Cast_Numeric_Columns to the input of this new target.
Click the "Amazon S3" node.
In the right panel, under "Node properties":
Node name: Transformed_Output.
Under "Data target properties - S3":
Format: CSV.
Compression Type: GZIP.
S3 Target Location: Browse to your s3://your-name-glue-bucket/ bucket.
Data Catalog update options: Select Do not update the Data Catalog.
File partitioning: Autogenerate files (Recommended).
Save and Run the Job:
Click "Save" at the top right of the Glue Studio page.
Click the "Run" button 

## Testing the Pipeline
Monitor Job Run: Go to the Glue console -> "Job runs" and monitor the status of name_etl_job_auto. It should eventually succeed.

Check Output in S3:
Navigate to your s3://your-name-glue-output-bucket/ bucket.
You should find a new folder containing part-r-xxxxx.gz files.
Download and decompress one of these .gz files to verify that it's a CSV with your transformed data (correct columns, types, and data rows).
