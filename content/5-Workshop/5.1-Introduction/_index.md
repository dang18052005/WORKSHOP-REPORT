---
title: "Introduction and Architecture"
date: "2026-10-03"
weight: 1
chapter: false
pre: " <b> 5.1 </b> "
---
## Overview

In an e-commerce application, placing an order triggers several downstream tasks: recording the order, sending a confirmation email, and notifying administrators. If all of these are handled within the same request (synchronously), the system slows down easily as order volume grows, and a failure in a secondary service can cause lost orders or duplicate emails.

This workshop builds an order-processing flow that is **event-driven, fault-tolerant, and observable** on AWS, then runs measurements to compare it against a synchronous approach.

## What you will do

- Decouple order intake from order processing using **Amazon SQS** and a dead-letter queue (**DLQ**)
- Prevent duplicate processing with **Amazon DynamoDB** (idempotency)
- Orchestrate the email and notification workflow with **AWS Step Functions**, **Amazon SES**, and **Amazon SNS**
- Monitor the system and receive alerts with **Amazon CloudWatch**
- Simulate failures and load to evaluate the effectiveness of the architecture

## Architecture
**Architecture diagram**
![Architecture](/images/5-Workshop/5.1-Introduction/002.png)

**Diagram explanation**

1. **Order intake.** The customer sends an order request to Amazon API Gateway. The `receive-order` Lambda function validates the order, puts it into the SQS queue, and immediately returns a 202 response, so the customer does not wait for the downstream tasks.
2. **Background processing.** SQS holds the order as a buffer when order volume grows. The `process-order` Lambda function takes the order from the queue, writes it to DynamoDB with a conditional write (an existing order ID is skipped) so each order is processed only once, and then starts Step Functions.
3. **Orchestration and notification.** Step Functions invokes the `send-email` Lambda function to send the confirmation email through Amazon SES, retrying up to 3 times on failure. When the workflow ends, Amazon SNS notifies the administrator.
4. **Error handling.** An order that has been received more than 3 times and still fails is moved by SQS to the dead-letter queue (DLQ). It is not lost and can be moved back to the main queue after the cause is fixed.
5. **Monitoring.** Amazon CloudWatch collects logs and metrics. When the DLQ has a message, Step Functions fails, or Lambda errors, an alarm triggers SNS to send an alert to the administrator.
6. **Baseline path.** The `POST /orders-sync` path processes orders synchronously: a single Lambda function writes to DynamoDB and sends the email inside the request. This path is used only for comparison in the experiments.

| Component | Role | Problem it addresses |
|---|---|---|
| SQS | Buffers and holds orders | Slow responses, lost orders |
| DynamoDB | Conditional write by order ID | Duplicate processing and sending |
| Step Functions | Orchestrates and retries each step | Transient failures in the email step |
| DLQ | Keeps failed orders | Lost orders when failures persist |
| CloudWatch and SNS | Monitoring and alerting | Slow incident detection |
## Evaluation metrics

- Response time of the order API (average, p95)
- Rate of orders lost when failures occur
- Number of duplicate emails sent
- Time from when a failure occurs until an administrator receives an alert

## Duration and cost

Approximately 2–3 hours. Most services fall within the free tier or incur very low costs at test scale. Be sure to perform the cleanup step at the end to avoid incurring charges.