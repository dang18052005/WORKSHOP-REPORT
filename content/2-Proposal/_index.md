---
title : "Proposal"
date : 2026-11-11
weight : 2
chapter : false
pre : " <b> 2. </b> "
---

# Design and Implementation of a Fault-Tolerant, Observable Event-Driven Order Processing Architecture on AWS, with Experimental Evaluation

### 1. Executive Summary

This project designs and implements an *event-driven* order processing flow on AWS. The system separates order intake from the downstream tasks (recording the order, sending a confirmation email, notifying the administrator) using an *Amazon SQS* queue and a dead-letter queue (*DLQ*), prevents duplicate processing with *Amazon DynamoDB*, orchestrates email and notification steps with *AWS Step Functions*, *Amazon SES* and *Amazon SNS*, and monitors the system with *Amazon CloudWatch*.

After deployment, the project simulates failures and load to measure the effectiveness of the architecture, then compares it with a synchronous processing approach.

### 2. Problem Statement

#### 2.1. Background

In an e-commerce application, placing an order is more than writing a record to a database. After the customer checks out, the system must also save the order, send a confirmation email, notify the administrator, and possibly do more (update inventory, create a shipment). These tasks depend on different services with different latency and reliability.

The simplest approach is to do everything inside a single request (synchronously): the API receives the order, calls each service in turn, and only then returns the result to the customer. This is easy to write but shows many limitations when order volume grows or when a secondary service fails.

#### 2.2. Specific Problems

- *Slow under load:* the response time of the order API depends on the slowest task in the chain. The customer waits even for work that does not need to be waited for (such as sending an email).
- *Orders are easily lost:* a failure in a secondary service (for example, the email fails to send) can fail the whole request, while the order may already be partially written.
- *Inconsistent results:* the API returns an error to the customer, but the order is already in the database. The customer retries and creates a second order.
- *Duplicate sends:* when the system retries after a failure, the same order may be processed or emailed more than once.
- *Hard to detect incidents:* without monitoring, administrators only find out about failures when customers complain.
- *Hard to scale in parts:* all steps run in the same place, so the capacity of each step cannot be tuned independently.

| Problem | Business impact | Measurable symptom |
|---|---|---|
| Slow responses | Customers abandon carts, poor experience | Average and p95 response time grow with load |
| Orders lost on failure | Lost revenue and reputation | Order loss rate under simulated failures |
| Duplicate processing or sending | Customers receive repeated emails, wrong data | Number of duplicate emails for the same order |
| Slow incident detection | Incidents last without anyone knowing | Time from failure to alert |

#### 2.3. Why an Event-Driven Approach

An event-driven architecture separates order intake from downstream processing with an intermediate queue. The API only needs to put the order into the queue and respond immediately. The remaining steps run in the background, can be retried independently, and failed orders are kept instead of disappearing. The trade-off is more complexity and two points that need care: a standard queue may deliver a message more than once, and processing results become asynchronous. For this reason the project adds an idempotency mechanism and monitoring.

#### 2.4. Project Objectives

1. Design an asynchronous order processing architecture on AWS that can be deployed as code (AWS SAM).
2. Ensure orders are not lost when a processing step fails, and are not processed or emailed twice.
3. Provide monitoring and alerting so administrators learn about incidents in time.
4. Evaluate experimentally and compare with the synchronous approach under the same conditions.
5. Write step-by-step documentation so others can reproduce the work.

### 3. Solution Architecture

#### 3.1. Overall Architecture

Main flow: *Receive order → Enqueue → Process → Orchestrate notifications → Monitor*.

| Component | AWS service | Role |
|---|---|---|
| Order intake | Amazon API Gateway, AWS Lambda | Receives order requests and puts the order into the queue, returning a fast response to the user |
| Queue | Amazon SQS | Decouples intake from processing and buffers when order volume grows |
| Dead-letter queue | SQS Dead-Letter Queue (DLQ) | Keeps orders that fail after the allowed number of retries, so no order is lost |
| Duplicate prevention | Amazon DynamoDB | Stores processed order IDs (idempotency) so each order is processed only once |
| Workflow orchestration | AWS Step Functions | Orchestrates the confirmation email and notification steps with retry logic |
| Email and notifications | Amazon SES, Amazon SNS | Sends confirmation emails to customers and notifications to administrators |
| Monitoring | Amazon CloudWatch | Tracks metrics and logs and sets alarms |

The whole system is deployed in the **Asia Pacific (Singapore) ap-southeast-1** region using AWS SAM.

#### 3.2. Fault-Tolerance Mechanisms

- *Decoupling with SQS:* orders are stored in the queue before processing, so a failure in a later step does not lose the order.
- *Retries and DLQ:* failed orders are retried a configured number of times; if they still fail, they are moved to the DLQ for inspection and reprocessing.
- *Idempotency with DynamoDB:* before processing, the system checks the order ID in the table; an order that has already been processed is not processed again.
- *Orchestration with Step Functions:* each step (email, notification) has its own state and retry logic, making each run easy to trace.
- *Partial batch response:* when a batch of messages contains failures, only the failed messages return to the queue and the rest are deleted.

#### 3.3. Monitoring and Alerting

CloudWatch tracks the main metrics: queue length, number of messages in the DLQ, number of Lambda errors, processing time, and number of failed Step Functions executions. When a threshold is exceeded (for example, the DLQ has a message), the system sends an alert to the administrator through SNS.

| Alarm | Condition | Meaning |
|---|---|---|
| DLQ not empty | At least 1 message in the DLQ | An order failed completely and needs attention |
| Step Functions failed | At least 1 failed execution in 1 minute | The email or notification workflow failed after all retries |
| Order Lambda errors | At least 1 error in 1 minute | The order processing function hit an error and the order will be retried |

In addition to alarms, a CloudWatch dashboard shows queue depth, DLQ depth, Lambda processing time, Step Functions executions, and API latency.

#### 3.4. Experiment Plan

After deployment, the project runs the following scenarios and compares them with the synchronous approach:

| Scenario | Purpose |
|---|---|
| Gradually increase order volume (load test) | Measure the order API response time (average, p95) |
| Simulate a failure in the processing step | Measure the order loss rate on failure and check that failed orders go to the DLQ |
| Resend the same order multiple times | Count duplicate emails and verify idempotency |
| Simulate an email service failure | Measure the time from the failure to the administrator receiving an alert |

#### 3.5. Evaluation Metrics

- Order API response time (average, p95)
- Order loss rate on failure
- Number of duplicate emails
- Time from failure to administrator alert

#### 3.6. Detailed Processing Flow

1. The customer sends an order request to API Gateway; a Lambda function validates the data, puts the order into SQS, and immediately returns **202 (Accepted)**.
2. The processing Lambda receives the message from SQS, writes an idempotency key to DynamoDB with a conditional write (each `orderId` can be written only once), saves the order, and starts the state machine.
3. Step Functions invokes the email function through Amazon SES, retrying up to 3 times with increasing wait times.
4. On success: an "order processed" notification is sent to the administrator through SNS. On failure after all retries: an alert is sent through SNS and the execution ends in a failed state.
5. A message that cannot be processed after the allowed number of receives (`maxReceiveCount = 3`) is moved by SQS to the DLQ; a CloudWatch alarm detects it and notifies the administrator.

Key design decisions:

| Decision | Reason |
|---|---|
| Use a standard SQS queue | High throughput; a message can be delivered more than once (at-least-once), so duplicate protection is needed on the processing side |
| `maxReceiveCount = 3` | Allows retrying transient errors without letting a bad message loop forever |
| `VisibilityTimeout` of 60 seconds, greater than the maximum run time of the processing Lambda (30 seconds) | Prevents the message from reappearing while the function is still working |
| Idempotency key expires after 60 seconds, record is removed after 24 hours (TTL) | A worker that dies midway does not leave the order stuck forever |
| Separate synchronous path as a baseline (`/orders-sync`) | Provides a fair comparison under the same conditions |

#### 3.7. Technologies and Tools

| Group | Tools |
|---|---|
| Infrastructure as code | AWS SAM (AWS CloudFormation) |
| Runtime | AWS Lambda, Node.js 20, arm64 architecture |
| AWS services | API Gateway, SQS, DynamoDB, Step Functions, SES, SNS, CloudWatch |
| Command-line tools | AWS CLI, SAM CLI, PowerShell |
| Measurement tool | `loadtest.js` script that generates load and measures response time |
| Reporting | Hugo, GitHub Pages |

#### 3.8. Data and Interface (API)

| Method, path | Description |
|---|---|
| `POST /orders` | Asynchronous path: puts the order into SQS and returns 202 immediately |
| `POST /orders-sync` | Synchronous baseline path: saves the order and sends the email inside the request |

An order contains: `email` (recipient of the confirmation), `items` (a list of `sku`, `qty`), and `total`. The fields `simulateFailure` and `simulateEmailFailure` are used only to simulate failures in experiments and will be removed in real use.

| DynamoDB table | Key | Content |
|---|---|---|
| Orders | `orderId` | Order information and processing status |
| Idempotency | `orderId` | Duplicate-prevention key, with TTL |

#### 3.9. Security

- Each Lambda has only the permissions it needs on its own resources (least privilege); only `ses:SendEmail` uses `Resource: '*'` for simplicity in the test environment.
- Access keys and API keys are never committed to GitHub; AWS CLI profiles and environment variables are used instead.
- SES in sandbox mode sends only to verified addresses or the SES simulator mailbox.
- After finishing, delete the stack and disable the access key.

### 4. Expected Outcomes

- An order processing system running on AWS in an event-driven style, deployable with AWS SAM.
- Orders are not lost when a processing step fails: failed orders are kept in the DLQ and can be reprocessed.
- The same order is not processed or emailed twice thanks to the idempotency mechanism.
- Administrators receive alerts when the system has an incident, through CloudWatch and SNS.
- Experimental data comparing the event-driven architecture with the synchronous approach using the metrics in section 3.5.
- Step-by-step workshop documentation (preparation, SQS and DLQ, idempotency, Step Functions, monitoring, experiments, resource cleanup).

### 5. Duration and Cost

The workshop takes about 2 to 3 hours. Most services fall within the free tier or cost very little at test scale. A resource cleanup step must be performed at the end to avoid charges, and a cost alert (AWS Budgets) should be set up from the start.

### 6. Implementation Plan

| Phase | Work | Result |
|---|---|---|
| Week 9 | Prepare the account, IAM permissions and tools; design the architecture | Environment ready, design document |
| Week 10 | Implement SQS, DLQ and duplicate prevention with DynamoDB | Valid orders written once, failed orders reach the DLQ |
| Week 11 | Implement Step Functions, SES, SNS and CloudWatch monitoring | Email workflow with retries, alerts working |
| Week 12 | Run failure and load experiments, clean up resources, finish the report | Comparison data, completed report |

### 7. Cost Estimate

| Service | Assessment at test scale |
|---|---|
| Lambda, SQS, DynamoDB (on-demand), Step Functions, SNS, CloudWatch | Usually within the free tier or very low cost for a few hundred orders |
| Amazon SES | In sandbox mode, sends only to verified addresses or the SES simulator mailbox |
| API Gateway | Cost per request, very low at test load |

Exact figures should be checked with the AWS Pricing Calculator before running and verified with Cost Explorer after 1 to 2 days. A cost alert (for example, a 5 USD threshold) is set up from the start, and all resources are deleted in the cleanup step.

### 8. Risks and Mitigation

| Risk | Mitigation |
|---|---|
| SES sandbox limits sending to 1 email per second, causing bottlenecks in large load tests | Use the SES simulator mailbox, test at moderate load, increase retry counts, or request to leave the sandbox |
| Forgetting to delete resources causes charges | A dedicated cleanup section, AWS Budgets, and a Cost Explorer check |
| Leaking the access key or API key of the test system | Never commit them to GitHub, use `.gitignore`, delete or disable the key when finished |
| IAM permission errors during deployment | Prepare the workshop policy in advance and verify with `aws sts get-caller-identity` |
| Measurements distorted by Lambda cold starts | Warm up first and run each load level at least 3 times |

### 9. Scope and Limitations

- **In scope:** order intake, asynchronous processing, duplicate prevention, email sending, alerting, monitoring, and experiments comparing with the synchronous approach.
- **Out of scope:** payments, inventory management, user interface, end-user login and authorization.
- **Limitations:** experiments are small-scale, so results are indicative; orders in the DLQ must be moved back to the queue manually (`start-message-move-task`), with no automatic reprocessing yet, which is a direction for future work.

### 10. Success Criteria

- Valid orders are processed completely, with a confirmation email and an order record in DynamoDB.
- No order is lost under simulated failures: failed orders are in the DLQ or the workflow reports the error clearly.
- The same order ID submitted multiple times creates only one record and one email.
- Administrators receive an alert when there is an incident.
- Complete comparison data between the two approaches using the metrics in section 3.5.

### 11. References

1. [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
2. [Handling errors for an SQS event source in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)
3. [AWS Step Functions: error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
4. [Amazon SQS: available CloudWatch metrics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html)
5. [CloudWatch Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)