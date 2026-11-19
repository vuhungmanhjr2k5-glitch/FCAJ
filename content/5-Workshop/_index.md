---
title : "Workshop"
date : "2026-10-03"
weight : 5
chapter : true
pre : " <b> 5. </b> "
---

<!-- workshop-detail:v1:begin -->
## Workshop overview

This workshop builds an asynchronous order-processing system on AWS, with the Moc storefront as a demonstration application. A visitor selects a product, enters an email address and submits an order. The backend accepts the request, processes it, sends notifications and stores a status that the visitor can track.

The objective goes beyond sending an email. The system must distinguish accepted requests from completed orders, handle messages delivered more than once, and help an operator investigate failures. The workshop therefore combines queues, a database, orchestration and monitoring.

### Main components

| Component group | Responsibility |
|---|---|
| SQS and DLQ | Buffer incoming requests and retain messages that meet the redrive failure conditions |
| Lambda and DynamoDB | Process requests, control duplicates and store order status |
| Step Functions | Coordinate tasks and apply the configured Retry/Catch behavior |
| SES and SNS | Send customer confirmations and administrator notifications |
| CloudWatch | Collect logs and metrics, display dashboards and evaluate alarms |

### Demonstration and verification

Moc is an experimental storefront without payments, inventory management or real deliveries. Follow the same order ID through the website, DynamoDB, Step Functions execution and confirmation email. DONE indicates that the application workflow completed; inbox delivery requires a separate check.

The exercises progress from environment preparation to successful processing, duplicate requests, controlled failures and cleanup. Distinguish observed results from expected outcomes. A successful deployment screenshot alone does not demonstrate successful end-to-end processing.

An optional product-page screenshot can introduce the demonstration. There is no need to submit another order just to capture the storefront. Present processing evidence in the experiment sections.
<!-- workshop-detail:v1:end -->
![Illustration](/images/5-Workshop/102.png)
<!-- workshop-split:v1 -->

## Workshop contents

- [5.1 Introduction and Architecture](5.1-Introduction/)
- [5.2 Preparation (IAM, Region, Tools)](5.2-Prerequisite/)
- [5.3 Queues and DLQ](5.3-sqs-dlq/)
- [5.4 Preventing Duplicate Processing](5.4-Idempotency/)
- [5.5 Orchestration and Notifications](5.5-Orchestration/)
- [5.6 Monitoring](5.6-Monitoring/)
- [5.7 Fault and Load Experiments](5.7-Experiment/)
- [5.8 Clean up](5.8-Clean-Up/)
