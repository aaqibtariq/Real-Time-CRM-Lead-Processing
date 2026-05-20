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

## Technologies Used

- AWS Lambda
- Amazon API Gateway
- - Amazon S3
- Amazon SQS
- Amazon SNS
- Amazon CloudWatch
- Close CRM Webhooks
- Python (boto3)
- JSON Event Processing
  
# Architecture Components


CRM Source System
Close CRM
Generates lead creation events
Sends webhook payloads in real time
Source system for customer acquisition events
API Layer
Amazon API Gateway
Exposes public webhook endpoint
Receives incoming HTTP POST webhook requests
Routes requests to ingestion Lambda

Endpoint:

POST /crm
Ingestion Layer
AWS Lambda — crm_webhook_ingestion_lambda

Responsibilities:

Parses webhook payload
Extracts lead metadata
Writes raw event JSON into S3
Sends delayed processing message into SQS
Raw Storage Layer
Amazon S3 — Raw Bucket

Bucket:

crm-lead-pipeline-raw-aqib

Folder:

source/

Purpose:

Immutable raw event storage
Replay capability
Audit trail
Source-of-truth webhook archive
Asynchronous Processing Layer
Amazon SQS Delay Queue

Queue:

crm-lead-delay-queue

Purpose:

Introduces 10-minute delay before enrichment
Allows CRM ownership assignment completion
Decouples ingestion from enrichment
Smooths traffic spikes
Dead Letter Queue (DLQ)

Queue:

crm-lead-delay-dlq

Purpose:

Captures failed processing events
Supports retry/error investigation
Improves pipeline resiliency
Enrichment Layer
AWS Lambda — crm_lead_enrichment_lambda

Responsibilities:

Consumes delayed SQS messages
Reads raw lead event from S3
Retrieves public lookup metadata
Enriches lead information
Writes final enriched JSON to target S3
Sends SNS notifications
Public Lookup Layer
Public Amazon S3 Lookup Bucket

Lookup URL:

https://***.s3.us-east-1.amazonaws.com/{lead_id}.json

Purpose:

Stores lead owner lookup data
Provides additional enrichment metadata
Simulates external lookup/enrichment service

Lookup attributes:

lead owner
lead email
funnel
lead metadata
Enriched Storage Layer
Amazon S3 — Enriched Bucket

Bucket:

crm-lead-pipeline-enriched-aqib

Folder:

target/

Purpose:

Stores enriched lead records
Final curated output layer
Downstream analytics/reporting source
Notification Layer
Amazon SNS

Topic:

crm-lead-email-alert-topic

Purpose:

Sends automated lead alert emails
Notifies stakeholders of newly enriched leads
Provides real-time business visibility

Email includes:

lead name
lead email
lead owner
funnel
status
creation date
Monitoring Layer
Amazon CloudWatch

Purpose:

Lambda logging
SQS monitoring
Error tracking
Debugging
Operational observability

Tracked metrics:

delayed messages
visible messages
failed executions
Lambda logs
DLQ activity

