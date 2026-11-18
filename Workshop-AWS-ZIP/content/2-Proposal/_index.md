---
title : "Proposal"
date : 2026-11-11
weight : 2
chapter : false
pre : " <b> 2. </b> "
---

# Design and Implementation of a Fault-Tolerant, Observable Event-Driven Order Processing Architecture on AWS, with Experimental Evaluation

### 1. Summary

This project designs and implements an *event-driven* order processing flow on AWS. The system separates order intake from the downstream tasks (recording the order, sending a confirmation email, notifying the administrator) using an *Amazon SQS* queue and a dead-letter queue (*DLQ*), prevents duplicate processing with *Amazon DynamoDB*, orchestrates the email and notification workflow with *AWS Step Functions*, *Amazon SES*, and *Amazon SNS*, and monitors the system with *Amazon CloudWatch*.

After deployment, the project simulates failures and load to measure the effectiveness of the architecture, and then compares it with a synchronous processing approach.

### 2. Problem

In an e-commerce application, the checkout step triggers many downstream tasks. If all of them are handled within the same request (synchronously), the system runs into the following problems:

- *Slow under load:* the response time of the order API depends on the slowest task in the processing chain.
- *Easy to lose orders:* a failure in an auxiliary service (for example, a failed email) can make the whole request fail.
- *Duplicates:* when the system retries after an error, the same order may be processed or emailed more than once.
- *Hard to detect incidents:* without monitoring, the administrator only learns about errors when customers complain.

Therefore, an architecture is needed that decouples the processing steps, tolerates transient failures, avoids duplicate processing, and provides timely alerts.

### 3. Proposed Solution

#### 3.1. Overall architecture

Main processing flow: *Receive order → Queue → Process → Orchestrate notifications → Monitor*.

| Component | AWS Service | Role |
|---|---|---|
| Order intake | Amazon API Gateway, AWS Lambda | Receives the order request and puts it on the queue, returning a fast response to the user |
| Queue | Amazon SQS | Decouples order intake from processing and buffers traffic when order volume increases |
| Dead-letter queue | SQS Dead-Letter Queue (DLQ) | Keeps orders that fail after the allowed number of retries, so no order is lost |
| Duplicate prevention | Amazon DynamoDB | Stores processed order IDs (idempotency) so that each order is processed only once |
| Workflow orchestration | AWS Step Functions | Orchestrates the confirmation email and notification steps, with retry logic |
| Email and notifications | Amazon SES, Amazon SNS | Sends the confirmation email to the customer and notifies the administrator |
| Monitoring | Amazon CloudWatch | Tracks metrics and logs and sets up alarms |

The whole system is deployed in the **Asia Pacific (Singapore) ap-southeast-1** region using AWS SAM.

#### 3.2. Fault-tolerance mechanisms

- *Decoupling with SQS:* the order is stored in the queue before processing, so a failure in a later step does not lose the order.
- *Retries and DLQ:* a failed order is retried a configured number of times; if it still fails, it is moved to the DLQ for inspection and reprocessing.
- *Idempotency with DynamoDB:* before processing, the system checks the order ID in the table; an order that has already been processed is not processed again.
- *Orchestration with Step Functions:* each step (email, notification) has its own state and retry logic, making each run easy to trace.

#### 3.3. Monitoring and alerting

CloudWatch tracks the key metrics: queue length, number of messages in the DLQ, number of Lambda errors, processing time, and number of failed Step Functions executions. When a threshold is exceeded (for example, the DLQ contains a message), the system sends an alert to the administrator through SNS.

#### 3.4. Experiment plan

After deployment, the project runs the following scenarios and compares them with the synchronous processing approach:

| Scenario | Purpose |
|---|---|
| Gradually increasing order volume (load test) | Measure the response time of the order API (average, p95) |
| Simulated failure in the processing step | Measure the rate of lost orders when errors occur and check that failed orders go to the DLQ |
| Resubmitting the same order multiple times | Count duplicate emails and verify the idempotency mechanism |
| Simulated failure of the email service | Measure the time from when an error occurs until the administrator receives an alert |

#### 3.5. Evaluation metrics

- Response time of the order API (average, p95)
- Rate of orders lost when errors occur
- Number of duplicate emails sent
- Time from when an error occurs until the administrator receives an alert

### 4. Expected Results

- An order processing system running on AWS in an event-driven way, deployable with AWS SAM.
- No orders are lost when a processing step fails: failed orders are kept in the DLQ and can be reprocessed.
- The same order is not processed or emailed more than once, thanks to the idempotency mechanism.
- The administrator receives an alert when the system has an incident, through CloudWatch and SNS.
- Experimental data comparing the event-driven architecture with the synchronous approach, using the metrics in section 3.5.
- A step-by-step workshop guide (preparation, SQS and DLQ, idempotency, Step Functions, monitoring, experiments, resource clean-up).

### 5. Duration and Cost

The workshop takes about 2 to 3 hours. Most services are within the free tier or cost very little at experimental scale. The clean-up step at the end must be completed to avoid incurring charges, and a cost alert (AWS Budgets) should be set up from the start.