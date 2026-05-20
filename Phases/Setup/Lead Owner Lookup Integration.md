
1. Receive SQS message after 10 minutes
2. Read raw CRM file from your raw S3 bucket
3. Get lead_id
4. Call public lookup URL:
   https://***-lead-owner.s3.us-east-1.amazonaws.com/{lead_id}.json
5. Merge raw CRM data + public lookup data
6. Write final enriched JSON to target S3


# Updated Enrichment Lambda code

```

import json
import boto3
import urllib.request
from datetime import datetime, timezone
from urllib.error import HTTPError, URLError

s3 = boto3.client("s3")

ENRICHED_BUCKET = "crm-lead-pipeline-enriched-aqib"
ENRICHED_PREFIX = "target"

PUBLIC_LOOKUP_BASE_URL = "https://***-lead-owner.s3.us-east-1.amazonaws.com"

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

            enriched_payload = {
                "lead_id": lead_id,
                "display_name": lead_data.get("display_name"),
                "status_label": lead_data.get("status_label"),
                "date_created": lead_data.get("date_created"),

                "lead_email": lookup_json.get("lead_email"),
                "lead_owner": lookup_json.get("lead_owner"),
                "funnel": lookup_json.get("funnel") or lead_data.get(FUNNEL_FIELD),

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

        except Exception as e:
            print("=== ENRICHMENT ERROR ===")
            print(str(e))
            raise e

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Public lookup enrichment completed"
        })
    }

```

Expected output:
   -   s3://crm-lead-pipeline-enriched-aqib/target/enriched_lead_<lead_id>.json
