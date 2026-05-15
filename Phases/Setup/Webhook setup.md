# Account Creation

- Go to https://app.close.com/login/ and create free account
- Login
- Click Settings
- Click Developer
- Under API Keys -> Click New API Key
- Give any name and Click create API
- Save that API key

## API Key Creation – CRM Integration Setup

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Setup/setup_files/api%20key%20creation.png" width="750"/> </p>


# Setup Close Webhook Subscription

- Use your AWS API URL and Close API Key to get Close Subs
    - Check AWS Setup.md for API URL

```
curl -X POST https://api.close.com/api/v1/webhook/ \
  -H "Content-Type: application/json" \
  -u "<CLOSE_API_KEY>:" \
  -d '{
    "events": [
      {
        "action": "created",
        "object_type": "lead"
      }
    ],
    "url": "https://<api-id>.execute-api.us-east-1.amazonaws.com/crm",
    "verify_ssl": true
  }'

```
Close uses Basic Auth where your API key is the username and the password is empty.

You will get these
  -  webhook_id
  -  signature_key
  -  status

You can also find Webhook_ID in Close by following

- Click Settings
- Click Developer
- Under Webhooks -> Click Three dots and copy ID


Next Step — Test Real CRM Event

Now go into Close CRM and:

# Create a New Lead in CLose for Testing

- Example:
- Name: Test Lead AWS
- Email: test@gmail.com

- Once created:

- Go to:
  - Lambda → crm_webhook_ingestion_lambda → Monitor → CloudWatch Logs
- You should now see:
  - real Close webhook payload
  - headers
  - event body
  - lead information

- Expected output

```

{
  "event": {
    "action": "created",
    "object_type": "lead"
  },
  "data": {
    "id": "lead_xxxxx",
    "display_name": "Test AWS Lead",
    "date_created": "2026-05-13T..."
  }
}

```


