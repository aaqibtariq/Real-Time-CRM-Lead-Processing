
# Create Lambda first

- Go to AWS Console → Lambda → Create function
- Function name: crm_webhook_ingestion_lambda
- Runtime: Python 3.12
- Architecture: x86_64
- Create it
- Then paste this Test code and later you can use Actual Code once setup complete:

```python

import json

def lambda_handler(event, context):

    body = json.loads(event["body"])

    print("=== CLOSE WEBHOOK PAYLOAD ===")
    print(json.dumps(body, indent=2))

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Webhook received successfully"
        })
    }

```

- Or you can use actual code

```python

import json
import boto3
from datetime import datetime, timezone

s3 = boto3.client("s3")

RAW_BUCKET = "crm-lead-pipeline-raw-aqib"
RAW_PREFIX = "source"

FUNNEL_FIELD = "custom.cf_am3UgCUhyM5iNDtAPL84enDjUrZx1JsyVZ9uD9TbYwG"

def lambda_handler(event, context):
    try:
        print("=== RAW API GATEWAY EVENT ===")
        print(json.dumps(event, indent=2))

        # API Gateway sends Close payload as a string inside event["body"]
        body = json.loads(event.get("body", "{}"))

        crm_event = body.get("event", {})
        lead_data = crm_event.get("data", {})

        lead_id = crm_event.get("lead_id") or lead_data.get("id")

        if not lead_id:
            raise ValueError("lead_id not found in webhook payload")

        # Keep only the required/source webhook data for raw storage
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

        return {
            "statusCode": 200,
            "body": json.dumps({
                "message": "Close webhook received and stored in S3",
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

- Click Deploy


# Create API Gateway Endpoint

- Use this flow:
    - API Gateway → HTTP API → Lambda Integration → POST route /crm
- After deployment, your endpoint will look like:
    - https://<api-id>.execute-api.us-east-1.amazonaws.com/crm
- Go to:
    - API Gateway → Create API → HTTP API → Build
    - API name: crm-webhook-api
    - Integration type: Lambda
    - Lambda function: crm_webhook_ingestion_lambda
    - Method: POST
    - Resource path: /crm
    - Stage -> $default
    - Auto-deploy: enabled
- Create

# Test URL

Use Postman or AWS CloudShell:

```
curl -X POST https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/crm \
  -H "Content-Type: application/json" \
  -d '{"test":"hello","event":{"lead_id":"lead_test_123"}}'
```

You will get response like
```
{
  "message": "Webhook received successfully"
}

```

you can also verify Logs in Lmabda
- Lambda → crm_webhook_ingestion_lambda → Monitor → View CloudWatch Logs

