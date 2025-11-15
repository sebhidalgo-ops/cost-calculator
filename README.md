# AWS Cloud Cost Tracking Dashboard (CUR + Glue + Athena + Lambda + S3)

<img width="1911" height="1031" alt="Screenshot 2025-11-14 120823" src="https://github.com/user-attachments/assets/1713758f-35dd-4f96-87ea-15e7f4043978" />


This project turns AWS Cost & Usage Reports (CUR) into a simple cost dashboard hosted on S3. Only the services actually used in the project are documented here.

-	AWS Cost & Usage Reports (CUR)
-	Amazon S3 (data bucket for CUR)
-	AWS Glue Crawler + Glue Data Catalog
-	Amazon Athena
-	AWS Lambda
-	Amazon S3 (dashboard bucket for static website)


---


## 1. Architecture Overview


1.	CUR delivers detailed billing data to an S3 data bucket.
2.	A Glue Crawler scans the CUR files and creates tables in the Glue Data Catalog.
3.	Athena queries those tables to compute cost summaries.
4.	A Lambda function runs the Athena query and writes JSON output to a dashboard S3 bucket.
5.	An S3-hosted static HTML page loads the JSON and uses Chart.js to render the dashboard.

   
---


## 2. Services and IAM Policies

Placeholders used:


-	`ACCOUNT_ID` – your AWS account ID
-	`CUR_BUCKET_NAME` – S3 bucket for Cost & Usage Reports
-	`DASHBOARD_BUCKET_NAME` – S3 bucket for the dashboard
-	`GLUE_ROLE_NAME` – IAM role for Glue crawler (for example, `AWSGlueServiceRole-CUR`)
-	`LAMBDA_ROLE_NAME` – IAM role for Lambda (for example, `CostTrackingLambdaRole`)
-	`REGION` – AWS region (for example, `us-east-1`)


### 2.1 Amazon S3 – CUR Data Bucket


Bucket: `CUR_BUCKET_NAME`
<img width="1912" height="1041" alt="Screenshot 2025-11-14 120857" src="https://github.com/user-attachments/assets/f5916599-6aae-4056-bcd0-03f180806b1f" />

Purpose: Stores the raw CUR files (Parquet) from the Billing console.


Typical additional policy (optional, to limit read access to analytics roles):


```json
{
"Version": "2012-10-17",
"Statement": [
{
"Sid": "AllowAnalyticsReadCUR", "Effect": "Allow",
"Principal": {
"AWS": [
"arn:aws:iam::ACCOUNT_ID:role/GLUE_ROLE_NAME",
 
"arn:aws:iam::ACCOUNT_ID:role/LAMBDA_ROLE_NAME"
]
},
"Action": [
"s3:GetObject", "s3:ListBucket"
],
"Resource": [
"arn:aws:s3:::CUR_BUCKET_NAME",
"arn:aws:s3:::CUR_BUCKET_NAME/*"
]
}
]
}


Role in project: Source of all billing data.


2.2	AWS Glue Crawler + Glue Data Catalog
Database: cost_tracking_db Crawler: cost-tracking-crawler
Source path: s3://CUR_BUCKET_NAME/CURReports/ (or your actual CUR prefix) IAM role: GLUE_ROLE_NAME
Glue crawler role policy:
{
"Version": "2012-10-17",
"Statement": [
{
 
"Sid": "GlueAccessCURBucket", "Effect": "Allow",
"Action": [
"s3:GetObject", "s3:ListBucket",
"s3:GetBucketLocation"
],
"Resource": [
"arn:aws:s3:::CUR_BUCKET_NAME",
"arn:aws:s3:::CUR_BUCKET_NAME/*"
]
},
{
"Sid": "GlueDataCatalogAccess", "Effect": "Allow",
"Action": [
"glue:CreateDatabase", "glue:GetDatabase", "glue:GetDatabases", "glue:CreateTable", "glue:UpdateTable", "glue:GetTable",
"glue:GetTables", "glue:DeleteTable", "glue:GetPartition", "glue:GetPartitions"
 
],
"Resource": "*"
},
{
"Sid": "GlueLogging", "Effect": "Allow",
"Action": [ "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"
],
"Resource": "*"
}
]
}
Role in project: Reads CUR files from S3 and creates/updates tables in the Glue Data Catalog so Athena can query them.


2.3	Amazon Athena
Athena uses the AWS Glue Data Catalog (cost_tracking_db) as its metadata store and is invoked by Lambda.
There is no separate dedicated Athena role here; permissions are granted through the Lambda execution role.
Required Athena-related permissions (in Lambda role):
{
"Sid": "AthenaQueryExecution", "Effect": "Allow",
 
"Action": [
"athena:StartQueryExecution", "athena:GetQueryExecution", "athena:GetQueryResults"
],
"Resource": "*"
}
Role in project: Executes SQL over CUR tables to produce cost summaries.


2.4	AWS Lambda – Cost Summary Generator
Function: for example, CostTrackingLambda Role: LAMBDA_ROLE_NAME
Purpose:
•	Run Athena query against cost_tracking_db
•	Wait for completion
•	Transform results into JSON
•	Save data/top_services.json into the dashboard S3 bucket Lambda execution role policy:
{
"Version": "2012-10-17",
"Statement": [
{
"Sid": "AthenaQueries", "Effect": "Allow",
"Action": [
"athena:StartQueryExecution",
 
"athena:GetQueryExecution", "athena:GetQueryResults"
],
"Resource": "*"
},
{
"Sid": "GlueCatalogRead", "Effect": "Allow",
"Action": [
"glue:GetDatabase", "glue:GetDatabases", "glue:GetTable", "glue:GetTables"
],
"Resource": "*"
},
{
"Sid": "S3AccessCURAndDashboard", "Effect": "Allow",
"Action": [
"s3:GetObject", "s3:ListBucket", "s3:PutObject"
],
"Resource": [
"arn:aws:s3:::CUR_BUCKET_NAME",
 
"arn:aws:s3:::CUR_BUCKET_NAME/*",
"arn:aws:s3:::DASHBOARD_BUCKET_NAME", "arn:aws:s3:::DASHBOARD_BUCKET_NAME/*"
]
},
{
"Sid": "LambdaLogging", "Effect": "Allow",
"Action": [ "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"
],
"Resource": "*"
}
]
}
Role in project: Automation layer that refreshes the JSON used by the dashboard.


2.5	Amazon S3 – Dashboard Bucket
Bucket: DASHBOARD_BUCKET_NAME
Contents:
•	index.html (dashboard)
•	data/top_services.json (output from Lambda) Bucket policy (public read, optional):
{
 
"Version": "2012-10-17",
"Statement": [
{
"Sid": "PublicReadForWebsite", "Effect": "Allow",
"Principal": "*",
"Action": "s3:GetObject",
"Resource": "arn:aws:s3:::DASHBOARD_BUCKET_NAME/*"
}
]
}
CORS configuration example:
{
"CORSRules": [
{
"AllowedOrigins": ["*"],
"AllowedMethods": ["GET"],
"AllowedHeaders": ["*"]
}
]
}
Role in project: Hosts the static dashboard and serves the JSON data to the browser.


3.	End-to-End Flow
1.	Cost & Usage Reports (CUR) are delivered regularly to s3://CUR_BUCKET_NAME/CURReports/.
 
2.	The Glue Crawler runs, scans that prefix, and updates tables in cost_tracking_db.
3.	Lambda calls Athena to run an aggregation query over the CUR table (for example: top services by unblended cost).
4.	Lambda writes the summarized JSON to
s3://DASHBOARD_BUCKET_NAME/data/top_services.json.
5.	The S3-hosted index.html loads data/top_services.json and renders the charts using Chart.js.
# cost-calculator
