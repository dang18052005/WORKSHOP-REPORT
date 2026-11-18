---
title: "Tracking Orders with Amazon CloudWatch"
date: 2026-11-13
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

## The API Accepted the Order. Where Is It Now?

Hello everyone!

In an asynchronous system, a successful acceptance response does not explain whether an order has completed or is still waiting.

This is a question I want to explore while building an order processing workflow on AWS.

This article draws inspiration from an [AWS Open Source Blog post about serverless practices with Lambda Powertools](https://aws.amazon.com/blogs/opensource/simplifying-serverless-best-practices-with-lambda-powertools/), written by **Tom McCarthy** and published on **June 22, 2020**. It discusses structured logging, metrics, and tracing for application observability.

Here, I focus on logs and metrics using my own order processing examples. The data and scenarios are illustrative, not measured experimental results.

## 1. The Investigation Scenario

Suppose a client submits `ORDER-OBS-01`. The API returns an order identifier and an acceptance response, but the order remains incomplete.

The proposed processing flow is:

```text
Client
  |
  v
Intake API
  |
  v
SQS Queue
  |
  v
Lambda Worker
  |
  v
DynamoDB
```

Several explanations need to be distinguished:

- The request is queued but has not reached a consumer.
- The consumer is waiting on a processing step.
- A processing attempt failed and the work is being retried.
- The message moved to a dead-letter queue.
- Business processing finished, but the status query does not reflect it correctly.

A generic log such as `Processing failed` would provide little information about the affected order or failing step.

## 2. Designing Searchable Order Logs

Powertools Logger supports JSON logging, Lambda execution context, and correlation IDs. Implementations should control logged fields to limit sensitive data exposure.

Reference: [Powertools for AWS Lambda — Logger](https://docs.aws.amazon.com/powertools/python/latest/core/logger/).

For the workshop, I propose:

| Field | Purpose in the Design |
|-------|------------------------|
| orderId | Identify the business order. |
| correlationId | Connect steps within the same processing flow. |
| messageId | Identify the message being processed. |
| function_request_id | Distinguish individual Lambda executions. |
| service | Identify the component producing the log. |
| event | Describe a processing milestone, such as order_received or order_completed. |
| errorType | Classify a failure when one occurs. |

The order identifier can remain stable across retries while the Lambda invocation identifier changes. Recording both separates an order from its processing attempts.

An illustrative completion log is:

```json
{
  "level": "INFO",
  "service": "order-worker",
  "event": "order_completed",
  "orderId": "ORDER-OBS-01",
  "correlationId": "FLOW-OBS-01",
  "messageId": "MESSAGE-OBS-01",
  "processing_ms": 240,
  "end_to_end_ms": 1840
}
```

The timing values are synthetic examples.

In this design, the API includes `correlationId` in the message, and the consumer reads it for logging. Enabling CloudWatch does not automatically establish this business correlation convention.

When processing batches, each message must use its own logging context rather than identifiers left over from a previous record.

## 3. Investigating with CloudWatch Logs Insights

CloudWatch Logs Insights provides commands for selecting, filtering, and sorting log fields. Selecting relevant log groups and a narrow time range limits the data scanned.

Reference: [CloudWatch Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html).

For the JSON structure above, an illustrative query is:

```text
fields @timestamp, @log, service, event, orderId, correlationId, errorType
| filter orderId = "ORDER-OBS-01"
| sort @timestamp asc
| limit 200
```

The proposed investigation procedure is:

1. Open CloudWatch Logs Insights.
2. Select the API and consumer log groups.
3. Choose the period when the order was submitted.
4. Select Logs Insights QL and run the query.
5. Compare the processing milestones in the results.

The query assumes that the JSON fields are discovered correctly. If more than 200 records match, adjust the limit or narrow the time range.

A missing completion log alone does not prove that an order was lost. Query scope, logging configuration, and database state must also be checked.

## 4. What Should the Dashboard Track?

Individual order searches should be complemented by a view of overall processing health.

### Lambda Metrics

| Metric | Meaning |
|--------|---------|
| Errors | Invocations that end with a function error. |
| Duration | Function execution time. |
| Throttles | Invocation requests that are throttled. |

`Duration` does not include all the time an order spends waiting in the queue.

Reference: [AWS Lambda metrics](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics-types.html).

### SQS Metrics

| Metric | Meaning |
|--------|---------|
| ApproximateNumberOfMessagesVisible | Messages available for retrieval. |
| ApproximateNumberOfMessagesNotVisible | Received messages not yet deleted or whose visibility timeout has not expired. |
| ApproximateAgeOfOldestMessage | The age of the oldest unprocessed message represented by the metric. |

SQS values are approximate. Message-age reporting also has exceptions for repeatedly failing messages, so it is not an exact waiting-time measurement for every order.

Reference: [Available CloudWatch metrics for Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html).

### Interpreting the Signals Together

If backlog and message age rise together, I would investigate arrival rate, consumer capacity, and slow processing steps.

If `Errors` remains low while orders accumulate, I would also inspect concurrency limits, message states, and individual record failures.

These are investigation paths, not definitive diagnoses from a single chart.

## 5. Does a Successful Lambda Invocation Mean Every Order Succeeded?

Not necessarily.

With partial batch responses, a handler can return failed record identifiers without throwing an exception for the entire invocation. Invocation failures and individual order failures therefore need to be distinguished.

Reference: [Handling errors for an SQS event source in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).

For the workshop, I propose separate events:

- `order_processing_failed`: an order processing attempt failed.
- `order_completed`: business processing finished.
- `duplicate_operation_skipped`: an already-applied effect was skipped.

The number of error logs is not the number of permanently failed orders. An order may fail initially and complete after a retry.

The report should reconcile unique `orderId` values with their final states.

## 6. Measuring the Right Order Latency

For comparison with synchronous processing, I propose three separate measurements:

| Measurement | Start and End |
|-------------|---------------|
| API response latency | Client sends a request → client receives a response. |
| Worker processing time | Worker starts → worker finishes one processing attempt. |
| End-to-end completion time | Client sends the original request → order completes under the defined criteria. |

The example log's `processing_ms` and `end_to_end_ms` represent different measurements.

End-to-end measurement requires preserving the original start time across retries and defining how clock differences between components are handled.

If the client detects completion through polling, the measurement also includes polling delay. This method must be documented in the report.

This article does not yet provide experimental data supporting a performance conclusion.

## 7. The Lab I Want to Build

I propose testing with synthetic data:

1. Submit a normal order and locate its processing milestones.
2. Increase consumer processing time and observe backlog.
3. Introduce a transient failure and inspect retry logs.
4. Introduce a persistent failure and inspect the DLQ message.
5. Remove the failure, recover the message, and reconcile order state.

Evidence should include queries, relevant logs, dashboard screenshots, and an order-state table.

My goal is to start with an `orderId` and explain its current step, encountered failures, and final outcome.

## References

1. [AWS Open Source Blog — serverless practices with Powertools, Tom McCarthy, June 22, 2020](https://aws.amazon.com/blogs/opensource/simplifying-serverless-best-practices-with-lambda-powertools/).
2. [Powertools for AWS Lambda — Logger](https://docs.aws.amazon.com/powertools/python/latest/core/logger/).
3. [CloudWatch Logs Insights — Query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html).
4. [AWS Lambda — Types of metrics](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics-types.html).
5. [Amazon SQS — Available CloudWatch metrics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html).
6. [AWS Lambda — Handling errors for an SQS event source](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).