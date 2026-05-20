SQS delay queue to wait 10 minutes before enrichment processing

- Main Queue: crm-lead-delay-queue
- DLQ: crm-lead-delay-dlq
- Queue URL: https://sqs.us-east-1.amazonaws.com/********/crm-lead-delay-queue

# SQS – Queue 

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/Queues.png" width="750"/> </p>

## Create DLQ first

- Go to:
    - AWS Console → SQS → Create queue
    - Type: Standard
    - Name: crm-lead-delay-dlq
    - Keep defaults and create queue.
 ## Main Delay Queue

 - SQS → Create queue
 - Type: Standard
 - Name: crm-lead-delay-queue

- Set
   -  Delivery delay: 10 minutes
    - Visibility timeout: 2 minutes
   -  Message retention period: 4 days
 
# Attach DLQ to Main Queue

- Open crm-lead-delay-queue
- Click Dead-letter queue
- Edit
- Visibility timeout - 2 min
- Delivery delay - 10 min
- Message retention period 4 days
- Redrive allow policy allow
- Redrive permission allow all
- Dead-letter queue enabled
- Set this queue to receive undeliverable messages. enabled
- Choose queue select your crm-lead-delay-dlq
- Maximum receives 3
- Save

## Amazon SQS – Delay Queue 

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/Edit%20crm-lead-delay-queue.png" width="750"/> </p>

## Amazon SQS – Dead Letter Queue (DLQ)

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/Dead-letter%20queue.png" width="750"/> </p>

# Copy Main Queue URL\

- After queue is created, open:
- crm-lead-delay-queue
- Copy the Queue URL.
- It will look like:
    - https://sqs.us-east-1.amazonaws.com/<account-id>/crm-lead-delay-queue
- We will use this in the ingestion Lambda

## Add SQS Permission to Lambda Role 

- Go to:
    - Lambda → crm_webhook_ingestion_lambda → Configuration → Permissions
- Click the Execution role.
- Add inline policy:

```

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSendMessageToLeadDelayQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": "arn:aws:sqs:us-east-1:********:crm-lead-delay-queue"
    }
  ]
}

```

- Create

# Updated Ingestion Lambda Code

```
import json
import boto3
from datetime import datetime, timezone

s3 = boto3.client("s3")
sqs = boto3.client("sqs")

RAW_BUCKET = "crm-lead-pipeline-raw-aqib"
RAW_PREFIX = "source"

SQS_QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/********/crm-lead-delay-queue"

FUNNEL_FIELD = "custom.cf_am3UgCUhyM5iNDtAPL84enDjUrZx1JsyVZ9uD9TbYwG"


def lambda_handler(event, context):
    try:
        print("=== RAW API GATEWAY EVENT ===")
        print(json.dumps(event, indent=2))

        body = json.loads(event.get("body", "{}"))

        crm_event = body.get("event", {})
        lead_data = crm_event.get("data", {})

        lead_id = crm_event.get("lead_id") or lead_data.get("id")

        if not lead_id:
            raise ValueError("lead_id not found in webhook payload")

        output_payload = {
            "subscription_id": body.get("subscription_id"),
            "event": {
                "id": crm_event.get("id"),
                "date_created": crm_event.get("date_created"),
                "date_updated": crm_event.get("date_updated"),
                "organization_id": crm_event.get("organization_id"),
                "user_id": crm_event.get("user_id"),
                "request_id": crm_event.get("request_id"),
                "object_type": crm_event.get("object_type"),
                "object_id": crm_event.get("object_id"),
                "lead_id": lead_id,
                "action": crm_event.get("action"),
                "changed_fields": crm_event.get("changed_fields", []),
                "meta": crm_event.get("meta", {}),
                "data": {
                    "id": lead_data.get("id"),
                    "display_name": lead_data.get("display_name"),
                    "date_created": lead_data.get("date_created"),
                    "date_updated": lead_data.get("date_updated"),
                    "status_label": lead_data.get("status_label"),
                    "status_id": lead_data.get("status_id"),
                    "created_by": lead_data.get("created_by"),
                    "created_by_name": lead_data.get("created_by_name"),
                    "updated_by": lead_data.get("updated_by"),
                    "updated_by_name": lead_data.get("updated_by_name"),
                    "contact_ids": lead_data.get("contact_ids", []),
                    FUNNEL_FIELD: lead_data.get(FUNNEL_FIELD)
                },
                "previous_data": crm_event.get("previous_data", {})
            },
            "ingestion_metadata": {
                "ingested_at_utc": datetime.now(timezone.utc).isoformat(),
                "source": "close_crm_webhook",
                "api_gateway_request_id": event.get("requestContext", {}).get("requestId")
            }
        }

        file_name = f"crm_event_{lead_id}.json"
        s3_key = f"{RAW_PREFIX}/{file_name}"

        s3.put_object(
            Bucket=RAW_BUCKET,
            Key=s3_key,
            Body=json.dumps(output_payload, indent=2),
            ContentType="application/json"
        )

        print("=== S3 WRITE SUCCESS ===")
        print(f"s3://{RAW_BUCKET}/{s3_key}")

        sqs_message = {
            "lead_id": lead_id,
            "raw_bucket": RAW_BUCKET,
            "raw_key": s3_key,
            "event_id": crm_event.get("id"),
            "display_name": lead_data.get("display_name"),
            "date_created": lead_data.get("date_created"),
            "status_label": lead_data.get("status_label"),
            "funnel": lead_data.get(FUNNEL_FIELD)
        }

        sqs.send_message(
            QueueUrl=SQS_QUEUE_URL,
            MessageBody=json.dumps(sqs_message)
        )

        print("=== SQS MESSAGE SENT ===")
        print(json.dumps(sqs_message, indent=2))

        return {
            "statusCode": 200,
            "body": json.dumps({
                "message": "Close webhook received, stored in S3, and sent to SQS",
                "lead_id": lead_id,
                "s3_path": f"s3://{RAW_BUCKET}/{s3_key}"
            })
        }

    except Exception as e:
        print("=== ERROR ===")
        print(str(e))

        return {
            "statusCode": 500,
            "body": json.dumps({
                "message": "Webhook processing failed",
                "error": str(e)
            })
        }

```

**Check SQS → crm-lead-delay-queue → Monitoring**
**check CloudWatch logs for:**
**=== S3 WRITE SUCCESS ===**
**=== SQS MESSAGE SENT ===**


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

## AWS Lambda – SQS Lead Enrichment Function

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/SQS%20Enrichment%20Lambda.png" width="750"/> </p>

# Amazon CloudWatch – Webhook Ingestion Logs

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/cloudwatch%20crm_webhook_ingestion_lambda.png" width="750"/> </p>
