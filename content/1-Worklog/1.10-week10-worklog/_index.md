---
title: "Week 10 Worklog"
date: 2026-10-12
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

**Internship period:** 12/10/2026 - 17/10/2026

### Week 10 Objectives:

- Evaluate system behavior during message redelivery and processing failures.
- Refine idempotency handling and the recovery procedure for failed messages.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 12/10/2026 | Create a failure matrix covering duplicate messages, invalid input, timeouts, and transient processing failures. | Failure test plan |
| 13/10/2026 | Define idempotency keys by business operation and use conditional writes to control repeated and concurrent processing. | [DynamoDB condition expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html) |
| 14/10/2026 | Resubmit messages and inject failures between processing steps; inspect business effects, processing states, and recovery behavior. | Idempotency test record |
| 16/10/2026 | Simulate transient and persistent failures; observe retries, backlog, alarms, and movement to the dead-letter queue. | [SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) |
| 17/10/2026 | Resolve the simulated failure cause, perform a controlled redrive, and verify the resulting order states. | Recovery procedure notes |

### Week 10 Achievements:

- Developed scenarios for testing idempotency and recovery from interrupted processing.
- Recorded behavior for each scenario without treating at-least-once delivery as an exactly-once processing guarantee.
- Results to add: [Passed and failed cases, remaining issues, and the scope of verified business effects.]