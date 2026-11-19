---
title: "Project Proposal"
date: 2026-10-05
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## 1. Project Title

**Building a Fault-Tolerant and Observable Event-Driven Order Processing Workflow on AWS and Comparing It with Synchronous Processing.**

## 2. Background and Problem Statement

In an e-commerce application, accepting an order can trigger several operations, including input validation, order recording, state updates, and notifications.

In a synchronous implementation, the client waits for the operations included in the request path to finish before receiving a result. Slow processing or dependency failures can increase response time and cause request timeouts.

Resubmitting a request after a timeout can also produce duplicate processing if appropriate controls are absent. However, a timeout does not necessarily mean that the order was never processed: some operations may have completed before the connection was interrupted.

An event-driven architecture separates order acceptance from downstream processing through events or messages. This can allow requests to be accepted while processing components are slow and enable retries after transient failures.

The approach also introduces challenges, including duplicate messages, delayed state visibility, queue backlogs, and tracing work across multiple components.

This project proposes two implementations with equivalent business scope and evaluates their differences through controlled experiments.

## 3. Project Objectives

### 3.1. Overall Objective

Build an event-driven order processing model on AWS with failure handling and state visibility, then compare it with synchronous processing using experimental data.

### 3.2. Specific Objectives

- Implement order acceptance and status-query APIs.
- Build a synchronous implementation as the comparison baseline.
- Build an asynchronous implementation using a queue and an independent processing component.
- Design retries, dead-letter handling, and failed-message recovery.
- Use idempotency to control the effects of repeated requests or messages.
- Correlate logs across components to follow individual orders.
- Create dashboards and alarms for failures, backlogs, and unsuccessful messages.
- Measure latency, completion throughput, success rate, and recovery behavior.
- Identify the suitable conditions, limitations, and trade-offs of each architecture.

## 4. Implementation Scope

### 4.1. Included Functions

The system will use synthetic data to:

- Accept order creation requests.
- Validate required input fields.
- Perform simulated business operations with adjustable processing time.
- Record and update order states.
- Provide status queries.
- Record simulated notification outcomes.
- Test failure handling, duplicate processing, and recovery.

The proposed states are:

| State | Meaning |
|-------|---------|
| ACCEPTED | The order has been accepted through the selected durable intake mechanism. |
| PROCESSING | The order is being processed. |
| COMPLETED | The required business operations have finished. |
| FAILED | The order has been classified as failed under the processing rules. |

Messages in the dead-letter queue will be tracked separately for investigation and recovery. Dead-letter placement must be reconciled with business state rather than automatically treating every affected order as permanently failed.

### 4.2. Scope Limitations

- No real payment integration or real customer data.
- No complete e-commerce platform.
- Experiments will run in one AWS Region with documented resource limits.
- Production-scale readiness is not a project acceptance target.
- Conclusions will be limited to the business scope, configurations, and workloads tested.

## 5. Proposed Architecture and Solution

### 5.1. Synchronous Implementation

The proposed processing flow is:

**Client → API Gateway → Business-processing Lambda → DynamoDB → Response**

The function executes the business operations within the request path and returns a result after completion or failure.

This implementation provides the baseline for comparing response latency, completion time, and failure behavior.

### 5.2. Event-Driven Implementation

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

### 5.3. AWS Service Responsibilities

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

## 6. Fault Tolerance and Observability Design

### 6.1. Retries and Dead-Letter Handling

The system will distinguish transient failures from input or business errors that cannot be resolved through retries.

Configuration and validation will cover:

- Function timeout.
- Queue visibility timeout.
- Batch size and concurrency limits.
- Receive-count threshold before dead-letter placement.
- Partial batch failure reporting.
- Inspection, root-cause correction, and controlled message redrive.

Messages will not be repeatedly returned to processing without limits while the failure cause remains unresolved.

### 6.2. Idempotency and Consistency

SQS Standard may deliver a message more than once and does not guarantee strict ordering. Consumers must therefore handle redelivery without producing unintended repeated business effects.

The proposed controls include:

- Stable idempotency keys based on the order and business operation.
- Conditional writes to control concurrent processing.
- Separate states for in-progress and completed work.
- Recovery behavior for interruptions between processing steps.
- State-transition controls that prevent older messages from overwriting newer states.
- Tests for both duplicate delivery and failures occurring after a business effect has already happened.

Storing a deduplication key does not automatically guarantee exactly-once execution of every external side effect. These limitations will be stated in the report.

### 6.3. Logs, Metrics, and Alarms

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

## 7. Experimental Methodology

### 7.1. Comparison Principles

Both implementations will use the same business scope, input dataset, and definition of a completed order.

Resource configuration, concurrency limits, retry policies, source revisions, and execution conditions will be recorded for reproducibility.

Two timing measurements must be distinguished:

- **API response latency:** the time from the client sending a request to receiving its response.
- **Order completion time:** the time from the original request to the order reaching the defined terminal state.

A fast asynchronous acceptance response will not be treated as proof of faster business completion.

### 7.2. Experimental Scenarios

| Scenario | Evaluation Focus |
|----------|------------------|
| Steady load | Latency and throughput under a relatively stable arrival rate. |
| Burst load | Intake behavior, backlog growth, and queue-drain time. |
| Slow processing step | Effects on API responses and order completion time. |
| Transient failure | Retry behavior and completion after the failure is removed. |
| Persistent failure | Message isolation, alarms, and recovery through the DLQ. |
| Request or message resubmission | Control of duplicate business effects. |

### 7.3. Evaluation Metrics

- p50 and p95 API response latency.
- p50 and p95 completion time for completed orders.
- Unique orders completed per unit of time.
- Completed, failed, and unfinished order rates at the observation cutoff.
- Backlog size and queue-drain time.
- Recovery time after the failure cause is removed.
- Detected duplicate business effects.
- Experimental cost where sufficient allocation data is available.

Failed and unfinished orders will be reported separately rather than silently excluded from analysis.

Each scenario will be repeated. The report will document run count, duration, workload, and variation between runs.

## 8. Implementation Plan

Implementation will focus on the final four weeks, using the knowledge and experimental components prepared earlier.

| Period | Main Activities | Expected Output |
|--------|-----------------|-----------------|
| Week 9: 05/10–10/10/2026 | Finalize the architecture and integrate APIs, queues, consumers, and state storage. | Two runnable processing flows, architecture diagrams, and API specifications. |
| Week 10: 12/10–17/10/2026 | Refine idempotency, retries, DLQ handling, logs, and alarms; run failure tests. | Failure test records and recovery instructions. |
| Week 11: 19/10–24/10/2026 | Run workload scenarios, collect data, and compare the implementations. | Raw data, summary tables, and evaluation charts. |
| Week 12: 26/10–31/10/2026 | Review results and finalize documentation, evidence, and the report. | Source code, implementation guide, and final report. |

## 9. Expected Deliverables and Acceptance Criteria

### 9.1. Deliverables

- Source code for the synchronous and event-driven implementations.
- Architecture diagrams and processing-flow documentation.
- Deployment, configuration, and resource-cleanup instructions.
- Test data, tools, and experimental scenarios.
- Dashboards, log queries, and alarm configurations.
- Retry, DLQ, and idempotency test records.
- A comparison report supported by data and evidence.
- Vietnamese and English worklogs and summary documentation.

### 9.2. Acceptance Criteria

- Both implementations satisfy the agreed business scope.
- Order states and processing progress can be inspected by identifier.
- Failure behavior is tested through reproducible scenarios.
- Failed-message recovery is documented and exercised.
- Duplicate-processing controls are verified for the defined business effects.
- Comparisons distinguish response latency from completion time.
- Limitations, failed cases, and unresolved issues are explicitly recorded.

The project does not assume that event-driven processing is always superior. The expected outcome is an evidence-based understanding of when each approach is suitable and what trade-offs it introduces.

## 10. Cost, Risks, and Controls

### 10.1. Cost Management

Cost depends on the Region, request volume, Lambda execution time, SQS traffic, DynamoDB operations, and retained log volume.

Before load testing, the project will:

- Estimate cost for the proposed configuration and workload.
- Configure AWS Budgets and alert thresholds.
- Limit request rate and run duration.
- Set appropriate log retention.
- Monitor resources and clean up after experiments.

Budget alerts do not automatically stop all resources. The workload generator must have its own limits and stop conditions.

The experiments will not be assumed to be entirely free or covered by the Free Tier.

### 10.2. Risks and Mitigations

| Risk | Proposed Mitigation |
|------|---------------------|
| Duplicate or out-of-order messages | Idempotency, state-transition controls, and redelivery tests. |
| Successful data write followed by failed message publication | Select a reliable intake mechanism; evaluate an outbox and reconciliation. |
| Rapid backlog growth | Monitor message age, limit load, and adjust consumer capacity within budget. |
| Repeated retries increasing cost | Classify failures, configure thresholds, and isolate messages in the DLQ. |
| Unfair comparison | Align business operations, document configurations, and repeat measurements. |
| Difficulty identifying failure causes | Structured logs, correlationId, and order-based investigation procedures. |
| Sensitive information in logs or source code | Use synthetic data, limit log content, and keep credentials out of source code. |

## 11. References

- [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [Amazon CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [AWS Pricing Calculator](https://calculator.aws/)