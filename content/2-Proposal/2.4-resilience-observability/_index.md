---
title: "Fault Tolerance and Observability"
weight: 4
chapter: false
pre: " <b> 2.4. </b> "
---

## 1. Fault Tolerance and Observability Design

### 1.1. Retries and Dead-Letter Handling

The system will distinguish transient failures from input or business errors that cannot be resolved through retries.

Configuration and validation will cover:

- Function timeout.
- Queue visibility timeout.
- Batch size and concurrency limits.
- Receive-count threshold before dead-letter placement.
- Partial batch failure reporting.
- Inspection, root-cause correction, and controlled message redrive.

Messages will not be repeatedly returned to processing without limits while the failure cause remains unresolved.

### 1.2. Idempotency and Consistency

SQS Standard may deliver a message more than once and does not guarantee strict ordering. Consumers must therefore handle redelivery without producing unintended repeated business effects.

The proposed controls include:

- Stable idempotency keys based on the order and business operation.
- Conditional writes to control concurrent processing.
- Separate states for in-progress and completed work.
- Recovery behavior for interruptions between processing steps.
- State-transition controls that prevent older messages from overwriting newer states.
- Tests for both duplicate delivery and failures occurring after a business effect has already happened.

Storing a deduplication key does not automatically guarantee exactly-once execution of every external side effect. These limitations will be stated in the report.

### 1.3. Logs, Metrics, and Alarms

Structured logs will include:

- orderId.
- eventId or messageId.
- correlationId.
- Processing step.
- State and error category.
- Start time, end time, and processing duration.

Dashboards will focus on:

- Accepted requests and completed orders.
- Lambda errors, duration, and throttles.
- Queue backlog and oldest-message age.
- Dead-letter queue messages.
- Order completion time and failure rate.

SQS queue metrics will support trend monitoring. Exact order reconciliation will use unique identifiers and experimental records.
