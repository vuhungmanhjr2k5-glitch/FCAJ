---
title: "Architecture"
weight: 2
chapter: false
pre: " <b> 5.1.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

{{< figure src="/images/5-Workshop/5.1-Introduction/100.png" alt="Overall architecture" >}}

<!-- workshop-detail:v1:begin -->
## Reading the architecture

Follow one order through the diagram. The intake layer places it in SQS. A worker Lambda handles the message and starts Step Functions. DynamoDB supports duplicate control and order status. Step Functions calls SES, then records completion or follows its failure path.

| Stage | Path |
|---|---|
| Intake | API → SQS |
| Processing | Lambda → Step Functions → SES |
| Failed delivery | SQS → DLQ once the receive threshold is crossed |
| Data and observation | DynamoDB stores state, CloudWatch aggregates signals, SNS delivers notifications |

## Responsibilities

- **API:** validate and accept input; acceptance is not completed processing.
- **SQS:** buffer messages and support redelivery.
- **Lambda:** run the application's message-processing logic.
- **Step Functions:** manage task order, retries and error handling.
- **DynamoDB:** persist state for tracking and duplicate checks.

## Connection to the Moc storefront

The deployed execute-api website serves its interface through API Gateway, an asset-serving Lambda and a private S3 bucket. Order requests use a separate API Lambda before entering the existing queue. Distinguish the asset-serving function, API function and order-processing worker.

An email failure inside Step Functions does not automatically place the original SQS message in the DLQ. SQS redrive and workflow Catch handling apply at different stages.

Reuse the existing architecture diagram. If it illustrates only the source workflow, state that scope instead of presenting it as the complete storefront hosting architecture. No additional deployment is required for a diagram screenshot.
<!-- workshop-detail:v1:end -->

## Locate the two boundaries on the diagram

First follow the arrow from SQS to Lambda. A message is temporarily hidden while the worker handles it. When the invocation returns successfully, the SQS integration can delete the message from the source queue. In section 5.5 the worker finishes its part after starting Step Functions. The source queue can therefore be empty while a customer's email task is still running.

Next follow Step Functions to SES. If SES rejects a request, execution history records the task failure and the configured retries. After exhaustion the workflow writes `FAILED` and publishes to SNS. The named execution is a good starting point for this incident. A malformed JSON body can fail earlier, before the worker knows an order ID or starts an execution. The Lambda log and the DLQ identify that failure.

## Predict the last observable step

For one submitted order, establish whether the API enqueued a message, the worker started a named execution, and that execution reached a terminal state. Checking those boundaries in sequence narrows the search even when a dashboard aggregates many orders. Queue URLs and the state machine ARN belong to a particular deployment. Re-read them from the stack when moving to another environment.
