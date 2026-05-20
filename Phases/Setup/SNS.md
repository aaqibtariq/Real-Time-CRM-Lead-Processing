
# SNS

##  STEP 1 — Create SNS Topic
- Go to AWS Console → SNS
- Click Create topic
- Choose:
- Type: Standard

- Name: crm-lead-email-alert-topic
- Click Create topic

 ## STEP 2 — Add Email Subscription

- Open your topic
- Click Create subscription
- Fill:
  - Protocol: Email
  - Endpoint: your email
- Click Create subscription
-  Go to your email → click Confirm subscription

# Step 3 - Copy Topic ARN

# Step 4 — Add SNS Permission to Enrichment Lambda Role

- Go to:
    - Lambda → crm_lead_enrichment_lambda → Configuration → Permissions
- Click the Execution role.
- Add inline policy:

```
{
  "Sid": "AllowPublishLeadEmailAlert",
  "Effect": "Allow",
  "Action": [
    "sns:Publish"
  ],
  "Resource": "arn:aws:sns:us-east-1:****:crm-lead-email-alert-topic"
}

```
- Polci name - crm-lead-email-sns-policy

# Step 5 - Update enrichment Lambda code


```

import json
import boto3
import urllib.request
from datetime import datetime, timezone
from urllib.error import HTTPError, URLError

s3 = boto3.client("s3")
sns = boto3.client("sns")

ENRICHED_BUCKET = "crm-lead-pipeline-enriched-aqib"
ENRICHED_PREFIX = "target"

PUBLIC_LOOKUP_BASE_URL = "https://***-lead-owner.s3.us-east-1.amazonaws.com"

SNS_TOPIC_ARN = "arn:aws:sns:us-east-1:***:crm-lead-email-alert-topic"

FUNNEL_FIELD = "custom.cf_am3UgCUhyM5iNDtAPL84enDjUrZx1JsyVZ9uD9TbYwG"


def fetch_lookup_data(lead_id):
    lookup_url = f"{PUBLIC_LOOKUP_BASE_URL}/{lead_id}.json"

    print(f"Reading public lookup file: {lookup_url}")

    try:
        with urllib.request.urlopen(lookup_url, timeout=10) as response:
            lookup_data = json.loads(response.read().decode("utf-8"))

        return lookup_data, "found", lookup_url

    except HTTPError as e:
        if e.code == 404:
            print(f"Lookup file not found for lead_id: {lead_id}")
            return {}, "missing", lookup_url
        raise e

    except URLError as e:
        print(f"Lookup URL error: {str(e)}")
        raise e


def send_email_notification(enriched_payload):
    subject = f"New Lead Alert - {enriched_payload.get('display_name', 'Unknown Lead')}"

    message = f"""
New Lead Alert

Name: {enriched_payload.get('display_name')}
Lead ID: {enriched_payload.get('lead_id')}
Created Date: {enriched_payload.get('date_created')}
Label: {enriched_payload.get('status_label')}
Email: {enriched_payload.get('lead_email')}
Lead Owner: {enriched_payload.get('lead_owner')}
Funnel: {enriched_payload.get('funnel')}

Lookup Status: {enriched_payload.get('lookup_status')}
Enriched At: {enriched_payload.get('enrichment_metadata', {}).get('enriched_at_utc')}
"""

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=subject[:100],
        Message=message
    )

    print("=== EMAIL NOTIFICATION SENT VIA SNS ===")
    print(message)


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

            raw_response = s3.get_object(
                Bucket=raw_bucket,
                Key=raw_key
            )

            raw_json = json.loads(raw_response["Body"].read())

            crm_event = raw_json.get("event", {})
            lead_data = crm_event.get("data", {})

            lookup_json, lookup_status, lookup_url = fetch_lookup_data(lead_id)

            funnel = (
                lookup_json.get("funnel")
                or lead_data.get(FUNNEL_FIELD)
                or message_body.get("funnel")
                or "Unknown Funnel"
            )

            enriched_payload = {
                "lead_id": lead_id,
                "display_name": lead_data.get("display_name") or message_body.get("display_name"),
                "status_label": lead_data.get("status_label") or message_body.get("status_label"),
                "date_created": lead_data.get("date_created") or message_body.get("date_created"),

                "lead_email": lookup_json.get("lead_email"),
                "lead_owner": lookup_json.get("lead_owner"),
                "funnel": funnel,

                "lookup_status": lookup_status,
                "lookup_url": lookup_url,

                "enrichment_metadata": {
                    "enriched_at_utc": datetime.now(timezone.utc).isoformat(),
                    "source_raw_bucket": raw_bucket,
                    "source_raw_key": raw_key
                }
            }

            enriched_key = f"{ENRICHED_PREFIX}/enriched_lead_{lead_id}.json"

            s3.put_object(
                Bucket=ENRICHED_BUCKET,
                Key=enriched_key,
                Body=json.dumps(enriched_payload, indent=2),
                ContentType="application/json"
            )

            print("=== ENRICHED FILE WRITTEN ===")
            print(f"s3://{ENRICHED_BUCKET}/{enriched_key}")
            print(json.dumps(enriched_payload, indent=2))

            send_email_notification(enriched_payload)

        except Exception as e:
            print("=== ENRICHMENT ERROR ===")
            print(str(e))
            raise e

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Lead enrichment and email notification completed"
        })
    }


```
Expected output in email

```
New Lead Alert

Name:
Lead ID:
Created Date:
Label:
Email:
Lead Owner:
Funnel:

```
