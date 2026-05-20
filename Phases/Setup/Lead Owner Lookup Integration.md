
1. Receive SQS message after 10 minutes
2. Read raw CRM file from your raw S3 bucket
3. Get lead_id
4. Call public lookup URL:
   https://***-lead-owner.s3.us-east-1.amazonaws.com/{lead_id}.json
5. Merge raw CRM data + public lookup data
6. Write final enriched JSON to target S3


# Updated Enrichment Lambda code
