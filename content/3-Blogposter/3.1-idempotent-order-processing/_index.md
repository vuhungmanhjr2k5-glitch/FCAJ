---
title: "Order Processing with Lambda and SQS"
date: 2026-11-12
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

## Retrying Without Duplicate Effects

Hello everyone!

While exploring event-driven order processing, I considered a question: **what happens when the same order reaches a processing component twice?**

This article draws on an [AWS Compute Blog post about idempotency](https://aws.amazon.com/blogs/compute/handling-lambda-functions-idempotency-with-aws-lambda-powertools/), published on **April 20, 2022**, and written by **Jerome Van Der Linden and Dariusz Osiennik**. It introduces Powertools as a way to support safe repeated processing.

The scenarios and order data below are my own illustrations for the project. This is a learning article, not a report of completed deployment experiments.

## 1. One Order, Two Processing Attempts

Suppose the system receives `ORDER-DEMO-01`. A worker records an order confirmation but fails before completing its execution.

The message is subsequently processed again.

If every invocation repeats all business operations, the same order could produce two confirmation records or two notifications.

The question to investigate is: **which business operations had already completed before the failure?**

AWS documents that Lambda's SQS integration may process a record more than once. Lambda reads messages in batches and deletes them after successful processing. Consumers therefore need to tolerate repeated messages without unintended effects.

Reference: [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html).

## 2. What Does Idempotency Address?

An idempotent operation does not produce additional effects when repeated with the same input.

Powertools for AWS Lambda provides utilities for managing processing keys, states, and results. This protection has an expiration window that should match the expected retry period.

Reference: [Powertools — Idempotency](https://docs.aws.amazon.com/powertools/python/latest/utilities/idempotency/).

My illustrative message is:

```json
{
  "orderId": "ORDER-DEMO-01",
  "operation": "CONFIRM_ORDER",
  "correlationId": "TRACE-DEMO-01"
}
```

The proposed business key is:

```text
ORDER-DEMO-01#CONFIRM_ORDER
```

It identifies both the order and the operation:

| Illustrative Situation | Intended Interpretation |
|------------------------|-------------------------|
| Confirming the same order again | Check the existing operation result. |
| Confirming a different order | A new task. |
| Cancelling a confirmed order | A different operation requiring state-transition checks. |

This is an illustrative convention. An implementation must define the uniqueness and scope of its order identifiers and keys.

## 3. Checking Whether a Key Exists Is Not Enough

The AWS article highlights concurrent invocations and failures occurring after a business effect but before its deduplication result is recorded.

A simple check followed by execution may therefore be insufficient.

Reference: [Edge cases discussed in AWS Compute Blog](https://aws.amazon.com/blogs/compute/handling-lambda-functions-idempotency-with-aws-lambda-powertools/).

For my workshop, I want to test this sequence:

1. Invocation A starts confirming an order.
2. Invocation B receives the same work before A finishes.
3. A is interrupted.
4. A subsequent retry arrives.

The experiment needs to examine both concurrent-processing prevention and recovery from interrupted work. Blocking every later invocation could leave the order stuck.

External effects, such as notifications or payment calls, also need appropriate deduplication or reconciliation. A DynamoDB record alone does not prove exactly-once execution across the entire workflow.

## 4. Can One Failed Message Cause the Whole Batch to Retry?

Consider a batch containing three messages:

| Message | Simulated Result |
|---------|------------------|
| msg-demo-a | Success |
| msg-demo-b | Failure |
| msg-demo-c | Success |

By default, a batch-processing failure can make successful messages available again.

AWS supports partial batch responses to identify individual failed messages. For this example:

```json
{
  "batchItemFailures": [
    {
      "itemIdentifier": "msg-demo-b"
    }
  ]
}
```

This requires enabling `ReportBatchItemFailures` on the event source mapping and implementing the corresponding response logic.

An unhandled handler exception still fails the entire batch. Partial batch responses also do not replace idempotency.

Reference: [Handling errors for an SQS event source in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).

## 5. A Dead-Letter Queue Supports Investigation

Invalid message data may continue to fail despite repeated attempts.

SQS can move messages to a dead-letter queue through a redrive policy. The `maxReceiveCount` setting determines the receive-count threshold for this movement.

A DLQ isolates messages for investigation and recovery; it does not repair their underlying failure.

Reference: [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).

My proposed workshop procedure is:

1. Identify the failed message and associated order.
2. Inspect logs to locate the failing step.
3. Check which business effects have already occurred.
4. Correct the cause and test recovery on a small sample.
5. Reconcile outcomes before recovering the remaining messages.

## 6. Applying the Ideas to My Order Processing Project

The proposed experimental model is:

```text
Intake API
    |
    v
 SQS Queue -------> DLQ
    |
    v
Lambda Worker
    |
    v
DynamoDB: order states and idempotency data
```

Logs and metrics will support processing visibility.

This diagram focuses on the consumer. Reliable order acceptance and work publication require a separate design in the architecture section.

I propose the following experiments:

| Scenario | Question to Investigate |
|----------|-------------------------|
| Resubmit the same work | Are additional business effects produced? |
| Run concurrent processing attempts | Can both acquire the right to process? |
| Interrupt execution after a business step | Does the retry resume from the appropriate state? |
| Fail one message in a batch | Are successful messages processed again? |
| Recover a message from the DLQ | Is the resulting order state consistent? |

The synchronous implementation will be evaluated against the same correctness requirements. API acceptance time will also be measured separately from business completion time.

This article does not yet contain measurements establishing which approach is faster or less expensive.

## 7. My Perspective

I want the project to demonstrate how an order can be tracked and recovered to a final state, including when processing fails between steps.

A useful starting lab would use a few synthetic orders, deliberately inject failures, and record message deliveries, function executions, and actual business effects separately.

That comparison would help explain whether repeated processing attempts also produce repeated outcomes.

If you have implemented a similar workflow, how do you test failures between business operations?

## References

1. [AWS Compute Blog — foundational idempotency article, April 20, 2022](https://aws.amazon.com/blogs/compute/handling-lambda-functions-idempotency-with-aws-lambda-powertools/).
2. [AWS Lambda — Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html).
3. [Powertools for AWS Lambda — Idempotency](https://docs.aws.amazon.com/powertools/python/latest/utilities/idempotency/).
4. [AWS Lambda — Handling errors for an SQS event source](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).
5. [Amazon SQS — Using dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).