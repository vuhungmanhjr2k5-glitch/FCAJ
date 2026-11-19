---
title: "Cost, Risks and References"
weight: 7
chapter: false
pre: " <b> 2.7. </b> "
---

## 1. Cost, Risks, and Controls

### 1.1. Cost Management

Cost depends on the Region, request volume, Lambda execution time, SQS traffic, DynamoDB operations, and retained log volume.

Before load testing, the project will:

- Estimate cost for the proposed configuration and workload.
- Configure AWS Budgets and alert thresholds.
- Limit request rate and run duration.
- Set appropriate log retention.
- Monitor resources and clean up after experiments.

Budget alerts do not automatically stop all resources. The workload generator must have its own limits and stop conditions.

The experiments will not be assumed to be entirely free or covered by the Free Tier.

### 1.2. Risks and Mitigations

| Risk | Proposed Mitigation |
|------|---------------------|
| Duplicate or out-of-order messages | Idempotency, state-transition controls, and redelivery tests. |
| Successful data write followed by failed message publication | Select a reliable intake mechanism; evaluate an outbox and reconciliation. |
| Rapid backlog growth | Monitor message age, limit load, and adjust consumer capacity within budget. |
| Repeated retries increasing cost | Classify failures, configure thresholds, and isolate messages in the DLQ. |
| Unfair comparison | Align business operations, document configurations, and repeat measurements. |
| Difficulty identifying failure causes | Structured logs, correlationId, and order-based investigation procedures. |
| Sensitive information in logs or source code | Use synthetic data, limit log content, and keep credentials out of source code. |

## 2. References

- [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [Amazon CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [AWS Pricing Calculator](https://calculator.aws/)
