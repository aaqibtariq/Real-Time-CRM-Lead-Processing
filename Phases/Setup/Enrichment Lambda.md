# Create Enrichment Lambda

- Go to:
  - Lambda → Create function
  - Function name: crm_lead_enrichment_lambda
  - Runtime: Python 3.12
  - Architecture: x86_64
  - Create
 

# IAM permissions

```
SQS read/delete
S3 read from raw bucket
S3 write to enriched bucket
CloudWatch logs
```

-  Go to:
  -  Lambda → crm_lead_enrichment_lambda → Configuration → Permissions
-  Click the Execution role.
-  Add inline policy:

```json

   {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadFromDelayQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "arn:aws:sqs:us-east-1:********:crm-lead-delay-queue"
    },
    {
      "Sid": "AllowReadRawLeadEvents",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::crm-lead-pipeline-raw-aqib/source/*"
    },
    {
      "Sid": "AllowWriteEnrichedLeadEvents",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::crm-lead-pipeline-enriched-aqib/target/*"
    }
  ]
}
```

- Polciy name -> crm-lead-enrichment-lambda-policy
- Create policy



# Enrichment Lambda Code

```
import json
import boto3
from datetime import datetime, timezone

s3 = boto3.client("s3")

ENRICHED_BUCKET = "crm-lead-pipeline-enriched-aqib"
ENRICHED_PREFIX = "target"

FUNNEL_FIELD = "custom.cf_am3UgCUhyM5iNDtAPL84enDjUrZx1JsyVZ9uD9TbYwG"


def lambda_handler(event, context):

    print("=== SQS EVENT RECEIVED ===")
    print(json.dumps(event, indent=2))

    for record in event["Records"]:

        try:

            message_body = json.loads(record["body"])

            print("=== PARSED SQS MESSAGE ===")
            print(json.dumps(message_body, indent=2))

            lead_id = message_body["lead_id"]
            raw_bucket = message_body["raw_bucket"]
            raw_key = message_body["raw_key"]

            print(f"Reading raw file from s3://{raw_bucket}/{raw_key}")

            raw_response = s3.get_object(
                Bucket=raw_bucket,
                Key=raw_key
            )

            raw_json = json.loads(
                raw_response["Body"].read()
            )

            crm_event = raw_json.get("event", {})
            lead_data = crm_event.get("data", {})

            enriched_payload = {
                "lead_id": lead_id,
                "display_name": lead_data.get("display_name"),
                "status_label": lead_data.get("status_label"),
                "date_created": lead_data.get("date_created"),
                "funnel": lead_data.get(FUNNEL_FIELD),

                "enrichment_metadata": {
                    "enriched_at_utc": datetime.now(
                        timezone.utc
                    ).isoformat(),
                    "source_raw_bucket": raw_bucket,
                    "source_raw_key": raw_key
                }
            }

            enriched_file_name = f"enriched_lead_{lead_id}.json"

            enriched_s3_key = (
                f"{ENRICHED_PREFIX}/{enriched_file_name}"
            )

            s3.put_object(
                Bucket=ENRICHED_BUCKET,
                Key=enriched_s3_key,
                Body=json.dumps(
                    enriched_payload,
                    indent=2
                ),
                ContentType="application/json"
            )

            print("=== ENRICHED FILE WRITTEN ===")
            print(
                f"s3://{ENRICHED_BUCKET}/{enriched_s3_key}"
            )

        except Exception as e:

            print("=== ENRICHMENT ERROR ===")
            print(str(e))

            raise e

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Enrichment completed"
        })
    }
```


# Attach SQS Trigger to Enrichment Lambda

- Go to:
    - Lambda → crm_lead_enrichment_lambda
    - Add Trigger
    - Source: SQS
    - SQS Queue crm-lead-delay-queue
    - Batch Size 1
    - Enable Trigger Yes
    - Click add

What this will do

```

Close CRM
    ↓
API Gateway
    ↓
Lambda Ingestion
    ↓
S3 Raw Layer
    ↓
SQS Delay Queue (10 mins)
    ↓
Enrichment Lambda
    ↓
S3 Enriched Layer
    ↓
DLQ for failures
```
