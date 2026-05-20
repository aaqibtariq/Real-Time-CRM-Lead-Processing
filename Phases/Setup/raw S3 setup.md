# Create S3 bucket

-  Create bucket:
  -  crm-lead-pipeline-raw-name
-  Settings:
  -  Block all public access: ON
  -  Bucket versioning: Optional
  -  Default encryption: SSE-S3
-  Create
-  Folder structure we will use:
  -  source/
    - crm_event_lead_xxxxx.json


# Add S3 permission to Lambda role

-  Go to:
  -  Lambda → crm_webhook_ingestion_lambda → Configuration → Permissions
-  Click the Execution role.
-  Add inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowWriteWebhookEventsToS3",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::crm-lead-pipeline-raw-aqib/source/*"
    }
  ]
}

```


## Amazon S3 – CRM Raw Data Landing Zone

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/CRM%20S3%20Raw.png" width="750"/> </p>

