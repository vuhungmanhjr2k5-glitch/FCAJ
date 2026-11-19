---
title: "Solution Architecture"
weight: 3
chapter: false
pre: " <b> 2.3. </b> "
---

## 1. Proposed Architecture and Solution

### 1.1. Synchronous Implementation

The proposed processing flow is:

**Client → API Gateway → Business-processing Lambda → DynamoDB → Response**

The function executes the business operations within the request path and returns a result after completion or failure.

This implementation provides the baseline for comparing response latency, completion time, and failure behavior.

### 1.2. Event-Driven Implementation

The proposed logical flow is:

**Client → API Gateway → Intake Lambda → SQS → Processing Lambda → DynamoDB**

Supporting components include:

- **SQS dead-letter queue:** holds messages that exceed the configured processing-attempt threshold.
- **CloudWatch:** provides logs, metrics, dashboards, and alarms.
- **Status-query API:** exposes progress and results by order identifier.

The intake API returns an order identifier and an acceptance response after the request enters the designed durable intake mechanism. This response does not mean that all business operations have completed.

If intake requires both recording an order in DynamoDB and publishing a message, the design must handle cases where one operation succeeds and the other fails. One option under consideration is a transactional outbox: commit the order and a work record in one transaction, then forward work to the queue using a relay with retry and reconciliation mechanisms.

If the message is sent directly to SQS before the consumer creates the order record, the status-query contract must explain the interval during which the order is not yet visible in the database.

The final choice will be documented and covered by integration tests.

### 1.3. AWS Service Responsibilities

| Service | Proposed Responsibility |
|---------|-------------------------|
| Amazon API Gateway | Expose order creation and status-query APIs. |
| AWS Lambda | Validate requests, execute business operations, and update states. |
| Amazon SQS | Buffer work and separate intake rate from processing capacity. |
| Amazon SQS DLQ | Isolate messages that repeatedly fail processing. |
| Amazon DynamoDB | Store order states and idempotency data. |
| Amazon CloudWatch | Collect logs, monitor metrics, and trigger alarms. |
| AWS IAM | Apply least-privilege access between components. |

Additional routing or notification services will be introduced only when required by a specific use case, keeping the architecture within the evaluation scope.
