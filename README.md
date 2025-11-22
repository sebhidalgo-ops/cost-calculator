# AWS Cloud Cost Tracking Dashboard




![backup-project-diagram 3](https://github.com/user-attachments/assets/8fb58d7b-df26-4731-9b2e-48d7ded69012)

## The Problem

AWS costs can spiral quickly across multiple services and accounts. Teams often struggle to:
- Identify which services are driving costs
- Track spending trends over time
- Get real-time visibility without manual CSV downloads
- Make data-driven decisions about resource optimization

This project solves that by automating cost tracking and visualization using AWS native services - turning raw billing data into actionable insights.

---




![Dashboard Screenshot](https://github.com/user-attachments/assets/1713758f-35dd-4f96-87ea-15e7f4043978)

A serverless solution that transforms AWS Cost & Usage Reports (CUR) into an interactive cost dashboard hosted on S3. This project uses AWS native services to automate cost tracking and visualization.

## Features

- 📊 Interactive cost visualization with Chart.js
- 🔄 Automated data pipeline using AWS services
- 💰 Real-time cost tracking by service
- 🌐 Static web hosting on S3
- 🔍 SQL-based cost analysis with Athena

## Architecture

### Services Used

- **AWS Cost & Usage Reports (CUR)** - Detailed billing data source
- **Amazon S3** - Storage for CUR data and dashboard hosting
- **AWS Glue** - Data cataloging and crawling
- **Amazon Athena** - SQL queries on cost data
- **AWS Lambda** - Automation and data processing

### Data Flow

1. **CUR** delivers detailed billing data to an S3 data bucket
2. **Glue Crawler** scans the CUR files and creates tables in the Glue Data Catalog
3. **Athena** queries those tables to compute cost summaries
4. **Lambda** runs the Athena query and writes JSON output to a dashboard S3 bucket
5. **S3-hosted static HTML** page loads the JSON and uses Chart.js to render the dashboard

![Architecture Diagram](https://github.com/user-attachments/assets/f5916599-6aae-4056-bcd0-03f180806b1f)

## What I Learned

Building this serverless cost tracking dashboard taught me valuable lessons about AWS FinOps and serverless architecture:

**AWS Services & Integration:**
- **Cost & Usage Reports (CUR)**: How enterprises track detailed billing data across accounts and services
- **AWS Glue**: Automating data catalog management and running crawlers to keep cost data queryable
- **Amazon Athena**: Writing SQL queries to analyze spending patterns and identify cost optimization opportunities
- **AWS Lambda**: Serverless event-driven processing for automated report generation
- **S3 Storage Strategies**: Using Standard storage for active data and understanding when to archive

**Technical Skills:**
- **Data Pipeline Design**: Building automated ETL workflows using AWS native services
- **Serverless Architecture**: Creating fully automated solutions without managing servers
- **Data Visualization**: Using Chart.js to transform raw cost data into actionable insights
- **IAM & Security**: Configuring least-privilege policies across multiple services
- **Cost Optimization**: Understanding how query optimization and automation reduce operational expenses

**Business Understanding:**
This project helped me see how FinOps teams operate - it's not just about tracking costs, it's about giving teams the visibility they need to make smart decisions about resource allocation and optimization.

**My Cloud Approach:**
I documented this entire build process because I believe the best way to learn is to teach. If I can explain AWS cost management to my 10-year-old daughter, I can help any team understand their cloud spending.

---

---

## Configuration

### Placeholders

Replace these placeholders with your actual values:

- `ACCOUNT_ID` - Your AWS account ID
- `CUR_BUCKET_NAME` - S3 bucket for Cost & Usage Reports
- `DASHBOARD_BUCKET_NAME` - S3 bucket for the dashboard
- `GLUE_ROLE_NAME` - IAM role for Glue crawler (e.g., `AWSGlueServiceRole-CUR`)
- `LAMBDA_ROLE_NAME` - IAM role for Lambda (e.g., `CostTrackingLambdaRole`)
- `REGION` - AWS region (e.g., `us-east-1`)

---

## Setup Guide

### 1. Amazon S3 - CUR Data Bucket

**Bucket:** `CUR_BUCKET_NAME`

**Purpose:** Stores the raw CUR files (Parquet format) from the Billing console.

**Bucket Policy (Optional):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAnalyticsReadCUR",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::ACCOUNT_ID:role/GLUE_ROLE_NAME",
          "arn:aws:iam::ACCOUNT_ID:role/LAMBDA_ROLE_NAME"
        ]
      },
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::CUR_BUCKET_NAME",
        "arn:aws:s3:::CUR_BUCKET_NAME/*"
      ]
    }
  ]
}
```

### 2. AWS Glue Crawler + Data Catalog

**Database:** `cost_tracking_db`  
**Crawler:** `cost-tracking-crawler`  
**Source Path:** `s3://CUR_BUCKET_NAME/CURReports/`  
**IAM Role:** `GLUE_ROLE_NAME`

**Purpose:** Reads CUR files from S3 and creates/updates tables in the Glue Data Catalog so Athena can query them.

**Glue Crawler Role Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GlueAccessCURBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::CUR_BUCKET_NAME",
        "arn:aws:s3:::CUR_BUCKET_NAME/*"
      ]
    },
    {
      "Sid": "GlueDataCatalogAccess",
      "Effect": "Allow",
      "Action": [
        "glue:CreateDatabase",
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:CreateTable",
        "glue:UpdateTable",
        "glue:GetTable",
        "glue:GetTables",
        "glue:DeleteTable",
        "glue:GetPartition",
        "glue:GetPartitions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "GlueLogging",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Amazon Athena

**Purpose:** Executes SQL queries over CUR tables to produce cost summaries.

Athena uses the AWS Glue Data Catalog (`cost_tracking_db`) as its metadata store and is invoked by Lambda. Permissions are granted through the Lambda execution role.

**Required Athena Permissions (in Lambda role):**

```json
{
  "Sid": "AthenaQueryExecution",
  "Effect": "Allow",
  "Action": [
    "athena:StartQueryExecution",
    "athena:GetQueryExecution",
    "athena:GetQueryResults"
  ],
  "Resource": "*"
}
```

### 4. AWS Lambda - Cost Summary Generator

**Function:** `CostTrackingLambda`  
**Role:** `LAMBDA_ROLE_NAME`

**Purpose:**
- Run Athena query against `cost_tracking_db`
- Wait for query completion
- Transform results into JSON
- Save `data/top_services.json` to the dashboard S3 bucket

**Lambda Execution Role Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AthenaQueries",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults"
      ],
      "Resource": "*"
    },
    {
      "Sid": "GlueCatalogRead",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:GetTable",
        "glue:GetTables"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3AccessCURAndDashboard",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::CUR_BUCKET_NAME",
        "arn:aws:s3:::CUR_BUCKET_NAME/*",
        "arn:aws:s3:::DASHBOARD_BUCKET_NAME",
        "arn:aws:s3:::DASHBOARD_BUCKET_NAME/*"
      ]
    },
    {
      "Sid": "LambdaLogging",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5. Amazon S3 - Dashboard Bucket

**Bucket:** `DASHBOARD_BUCKET_NAME`

**Contents:**
- `index.html` - Dashboard web page
- `data/top_services.json` - Output from Lambda

**Purpose:** Hosts the static dashboard and serves the JSON data to the browser.

**Bucket Policy (Public Read - Optional):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::DASHBOARD_BUCKET_NAME/*"
    }
  ]
}
```

**CORS Configuration:**

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["*"],
      "AllowedMethods": ["GET"],
      "AllowedHeaders": ["*"]
    }
  ]
}
```

---

## Deployment Steps

1. **Enable Cost & Usage Reports**
   - Navigate to AWS Billing Console
   - Create a new CUR report with Parquet format
   - Configure delivery to `s3://CUR_BUCKET_NAME/CURReports/`

2. **Create Glue Database and Crawler**
   - Create database `cost_tracking_db` in Glue
   - Configure crawler with the CUR S3 path
   - Run the crawler to populate the catalog

3. **Deploy Lambda Function**
   - Create Lambda function with the provided role policy
   - Configure environment variables for bucket names
   - Set up a trigger (e.g., EventBridge schedule)

4. **Setup Dashboard Bucket**
   - Enable static website hosting
   - Upload `index.html`
   - Configure CORS and bucket policy

5. **Test the Pipeline**
   - Manually invoke Lambda function
   - Verify `data/top_services.json` is created
   - Access the dashboard URL

---

## Usage

Once deployed, the dashboard automatically updates based on your Lambda trigger schedule. Access the dashboard at:

```
http://DASHBOARD_BUCKET_NAME.s3-website-REGION.amazonaws.com
```

---

## Future Enhancements

This is v1 - here's what I'm considering for future iterations:

**Alerting & Notifications:**
- [ ] SNS/Email alerts when spending exceeds defined thresholds
- [ ] Slack integration for real-time cost anomaly notifications
- [ ] Daily/weekly cost summary reports

**Advanced Analytics:**
- [ ] Tag-based cost allocation by team, project, or environment
- [ ] Month-over-month and year-over-year cost comparisons
- [ ] Cost forecasting based on historical trends
- [ ] Service-specific recommendations for optimization

**User Experience:**
- [ ] Date range filters for custom time period analysis
- [ ] Export functionality (PDF/CSV reports)
- [ ] Multi-account cost aggregation for organizations
- [ ] Mobile-responsive dashboard design

**Automation:**
- [ ] Automated cost anomaly detection using CloudWatch
- [ ] Integration with AWS Cost Anomaly Detection service
- [ ] Scheduled reports via EventBridge

Feedback and suggestions welcome! Open an issue or connect with me on LinkedIn.

---

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## About Me

Hi, I'm **Sebastian Hidalgo** - transitioning into AWS cloud architecture after 12 years in financial services, where I was recognized as a top advisor nationally.

**Why "The Cloud Dad"?**

I bring the same approach to cloud that made me successful in financial services: making complicated things simple, building trust through clarity, and always leading by example.

If I can explain AWS to my kids, I can help any team understand their cloud infrastructure.

**My Journey:**
- 🎓 Building AWS expertise through hands-on projects and certifications
- 📺 Documenting everything on YouTube for others to learn from
- 💼 Working toward AWS Solutions Architect or Technical Account Manager roles
- 👨‍👧‍👦 Teaching cloud concepts the way I teach my kids - with patience, humor, and real-world examples

**Cloud Rinse Repeat**: *Learn it. Build it. Document it. Teach it.*

---

### Connect With Me

- 📺 **YouTube**: [The Cloud Dad](https://youtube.com/@theclouddad) - Cloud tutorials and project walkthroughs
- 💼 **LinkedIn**: [Sebastian Hidalgo](https://linkedin.com/in/sebastian-hidalgo-9123a8221) - Following my cloud journey
- 🐙 **GitHub**: [@sebhidalgo-ops](https://github.com/sebhidalgo-ops) - More projects and learning in public

---

### Questions? Feedback?

I'm always learning and improving. If you have questions about this project, suggestions for enhancements, or want to connect about cloud architecture - I'd love to hear from you!

- Open an issue on this repo
- Connect with me on LinkedIn
- Comment on my YouTube videos

**Thanks for checking out my work!** ☁️👨‍👧‍👦

---

*This project is part of my public learning journey as I transition from financial services to AWS cloud architecture. Follow along as I build, document, and share everything I learn.*
