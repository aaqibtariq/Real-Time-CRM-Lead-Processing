# Real-Time-CRM-Lead-Processing

## Objective

Build a real-time CRM lead processing pipeline that automatically captures new leads from Close CRM, stores raw lead events in Amazon S3, enriches lead data after a delayed processing window,
and sends automated notifications to the sales/business team.

The solution uses a fully serverless AWS architecture to support scalable, low-maintenance, event-driven lead processing.


## Core Objective


The core objective of this project is to design a production-style real-time event-driven data pipeline that:

- Receives CRM lead events instantly through webhooks
- Stores raw immutable lead events for auditability
- Delays processing to allow CRM owner assignment completion
- Enriches lead records using external lookup data
- Sends automated notifications with enriched lead details
- Demonstrates enterprise-grade serverless architecture patterns using AWS services


## Project Goal

The goal of this project is to simulate a real-world customer acquisition and sales lead processing platform where leads generated in a CRM system are automatically:

- Captured in real time
- Persisted in a raw data lake
- Processed asynchronously
- Enriched with additional metadata
- Delivered to stakeholders through automated notifications

This project focuses heavily on:

- event-driven architecture
- serverless design
- delayed asynchronous processing
- decoupled systems
- production monitoring
- failure handling using DLQ
- scalable cloud-native pipelines


## Abstract

This project implements a fully serverless real-time CRM lead processing system using AWS services including API Gateway, Lambda, S3, SQS, SNS, and CloudWatch.

New leads created inside Close CRM are pushed through webhook subscriptions into an API Gateway endpoint. The ingestion Lambda captures webhook payloads and stores raw lead events into Amazon S3
while simultaneously publishing metadata messages into an Amazon SQS delay queue.

A 10-minute delayed queue allows sufficient time for CRM lead ownership and downstream attributes to stabilize before enrichment begins. After the delay period, a second Lambda function consumes SQS messages, 
retrieves the raw lead payload, enriches the data using public S3 lookup files, stores enriched lead output into a target S3 bucket, and publishes automated email notifications through Amazon SNS.

The architecture demonstrates a scalable, fault-tolerant, loosely coupled serverless data engineering workflow commonly used in production customer data platforms and marketing automation systems.

## End-to-End System Design – Real-Time CRM Lead Processing

<p align="center"> <img src="https://raw.githubusercontent.com/aaqibtariq/Real-Time-CRM-Lead-Processing/main/Phases/Architecture/SD%20for%20CRM%20Lead%20processing.png" width="850"/> </p>


## Architecture Components


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

```
Close CRM
   ↓
API Gateway
   ↓
Ingestion Lambda
   ↓
Raw S3 Bucket
   ↓
SQS Delay Queue (10 min)
   ↓
Enrichment Lambda
   ↓
Public S3 Lookup
   ↓
Enriched S3 Bucket

```

SNS 

```

Close CRM
   ↓
API Gateway
   ↓
Ingestion Lambda
   ↓
Raw S3 Bucket
   ↓
SQS Delay Queue (10 min)
   ↓
Enrichment Lambda
   ↓
Public S3 Lookup
   ↓
Enriched S3 Bucket
   ↓
SNS Email Notification

```

- Close CRM webhook subscription
- API Gateway endpoint
- Ingestion Lambda
- Raw S3 storage
- SQS 10-minute delay queue
- DLQ retry/error handling
- Enrichment Lambda
- Public S3 lookup by lead_id
- Enriched S3 target output
- SNS email notification
