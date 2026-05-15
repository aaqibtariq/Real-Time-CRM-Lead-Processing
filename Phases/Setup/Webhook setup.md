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
