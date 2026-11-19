---
title : "Workshop"
date : "2026-10-03"
weight : 5
chapter : true
pre : " <b> 5. </b> "
---

<!-- workshop-detail:v1:begin -->
<!-- moc:tong-quan -->
# From an Order on Moc to a Confirmation Email {#workshop-overview}

A customer selects two mugs on the **Moc** website, enters their email, and clicks **Place Order**. Once the order ID appears on the screen, the customer can close the browser. On AWS, the system continues processing the order and sends a confirmation email. In this workshop, you will build this asynchronous processing workflow and use the `orderId` to trace each step.

## How Does an Order Flow Through the System?

1. **Ingestion.** The API validates the request, pushes the order into an Amazon SQS queue, and returns an HTTP `202` status code. At this point, Moc has accepted the order, and the customer receives an ID for tracking.
2. **Recording.** AWS Lambda reads the message and writes the order to Amazon DynamoDB using the `orderId`. If SQS redelivers the same message, Lambda uses this ID to recognize that the order already exists in the table.
3. **Email Delivery.** AWS Step Functions orchestrates the email-sending step via Amazon SES. Once SES accepts the email request, the workflow records the `DONE` status. If the sending step still fails after retries, the workflow records `FAILED` and alerts the operator via Amazon SNS.

## If the Customer Clicks Place Order a Second Time

Suppose the connection drops right after the first click. In the test case, you resend the same `orderId`, query the DynamoDB record, and open the Step Functions execution history. Compare the number of received API requests with the number of successful email-sending steps to see how the duplicate click was handled.

When multiple orders arrive concurrently, you will separately measure the **API response time** and the **time for an order to reach its final state**. If the gap between these two timestamps widens, check the SQS queue metrics in Amazon CloudWatch to see how many messages are waiting for Lambda to process.

{{< moc-tong-quan "style" >}}
<!-- /moc:tong-quan -->

### Key Components

| Component Group | Role in the System |
|---|---|
| SQS and DLQ | Stores pending requests; retains unprocessable messages according to the redrive policy |
| Lambda and DynamoDB | Processes requests, handles idempotency, and stores order states |
| Step Functions | Orchestrates workflow steps and executes Retry/Catch configurations |
| SES and SNS | Sends order confirmations to customers and alerts to administrators |
| CloudWatch | Monitors logs, metrics, dashboards, and alarms |

### Deliverables and Verification Criteria

The Moc website is a test storefront without real payment, inventory checking, or physical shipping. A test order must be cross-checked using the same order ID across the UI, DynamoDB, Step Functions executions, and the received email. The `DONE` status indicates that the application workflow has completed; whether the email lands in the Inbox must be verified separately.

The hands-on labs progress from environment preparation to happy-path testing, duplicate submissions, controlled fault injection, and resource cleanup. When documenting your report, distinguish between observed results and expected targets; do not use a single successful deployment screenshot as evidence that the entire end-to-end workflow succeeded.

![Architecture Overview](/images/5-Workshop/102.png)
<!-- workshop-detail:v1:end -->

<!-- workshop-split:v1 -->

## Hands-on Roadmap

- [5.1 Introduction and Architecture](5.1-Introduction/)
- [5.2 Prerequisites (IAM, Region, Tools)](5.2-Prerequisite/)
- [5.3 Queues and DLQ](5.3-sqs-dlq/)
- [5.4 Idempotency](5.4-Idempotency/)
- [5.5 Orchestration and Notifications](5.5-Orchestration/)
- [5.6 Monitoring](5.6-Monitoring/)
- [5.7 Fault and Load Experiments](5.7-Experiment/)
- [5.8 Clean Up](5.8-Clean-Up/)