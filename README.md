# Real-Time-CRM-Lead-Processing


- Create AWS Lambda Function (crm_webhook_ingestion_lambda)
- Add Initial Webhook Test Code in Lambda
- Create API Gateway HTTP API
- Configure POST /crm Route
- Enable $default Stage Auto Deployment
- Copy API Gateway Webhook URL
- Create Close CRM API Key
- Create Close CRM Webhook Subscription
- Test Real-Time Lead Creation from Close CRM
- Verify Webhook Payload in CloudWatch Logs
- Create Amazon S3 Raw Bucket (crm-lead-pipeline-raw-aqib)
- Create source/ Folder in S3
- Add S3 PutObject Permission to Lambda IAM Role
- Update Lambda to Parse CRM Webhook Payload
- Extract lead_id and Required Lead Fields
- Generate Dynamic File Name (crm_event_{lead_id}.json)
- Store CRM Event JSON into Amazon S3
- Validate File Creation in S3 Bucket
- Verify End-to-End Webhook Ingestion Flow


- raw S3 setup.md
- Webhook setup.md
- ingestion_lambda setup.md
- Close CRM → API Gateway → Lambda → S3 Raw Folder

- SQS Delay Queue.md

```
Close CRM
    ↓
API Gateway
    ↓
Lambda Ingestion
    ↓
S3 Raw Storage
    ↓
SQS Delay Queue (10 mins)
    ↓
DLQ (failure handling)
```
- Enrichment S3.md
- Enrichment Lambda.md
- Lead Owner Lookup Integration.md
