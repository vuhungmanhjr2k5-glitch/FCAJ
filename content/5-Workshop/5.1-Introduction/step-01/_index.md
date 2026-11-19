---
title: "What you will do"
weight: 1
chapter: false
pre: " <b> 5.1.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

- Decouple order intake from order processing using **Amazon SQS** and a dead-letter queue (**DLQ**)
- Prevent duplicate processing with **Amazon DynamoDB** (idempotency)
- Orchestrate the email and notification workflow with **AWS Step Functions**, **Amazon SES**, and **Amazon SNS**
- Monitor the system and receive alerts with **Amazon CloudWatch**
- Simulate failures and load to evaluate the effectiveness of the architecture

<!-- workshop-detail:v1:begin -->
## Expected deliverables

Explain the problem each service addresses, deploy the configuration and verify results using real observations. Resources appearing in the AWS Console are only a starting point; check how they work together.

| Activity | Expected outcome | Verification |
|---|---|---|
| Accept an order | A valid request is accepted and can be tracked | API response or storefront |
| Process an order | The workflow reaches an appropriate terminal state | Execution and DynamoDB record |
| Control duplicates | Repeated requests are recognized | Logs for the same order and task history |
| Handle failures | A failure is recorded with a defined handling path | Catch branch, FAILED state or DLQ, depending on the failure |
| Monitor operations | The operator identifies the affected resource and time | Dashboard, alarm history and notifications |

## Example scenario

A visitor orders two cups. The website receives an order ID after acceptance, while email processing continues in the background. If the result of the first submission is uncertain, a retry should preserve the request identity to reduce accidental creation of another order.

## Evaluation boundaries

Limit conclusions to the tested inputs and observations. A successful duplicate test does not establish exactly-once effects under every possible failure. Record the input, submission count, observation time and actual result.

The objectives table is sufficient here; new screenshots are optional. Capture execution evidence when completing the relevant deployment and experiment sections.
<!-- workshop-detail:v1:end -->

## Follow a purchase from checkout to its final state

Suppose a customer orders two cups and the browser submits `MOC-101`. The asynchronous API returns HTTP 202 once SQS accepts the message. The storefront can now show “Order MOC-101 received.” A Lambda invocation may still be waiting to start. The application can report `DONE` once the workflow submits the email and writes that status to the table.

Now let the connection drop just after checkout. A second click can produce a second SQS `MessageId` for the same purchase. Reusing the original `orderId` allows the worker to recognize the retry. A customer who actually wants another two cups needs a new order ID. This distinction affects what you count when you assess duplicate processing later.

## Match each question to an observation

| Reader | Question | Where to look |
|---|---|---|
| Customer | Has the system accepted my order? | The API response and its order ID |
| Support | How far has the order progressed? | The item under that key and the named workflow execution |
| Operator | Where can I investigate a failure? | The failing state, or the DLQ body if parsing failed at the worker |

At the end of the lab, write a three-line account for one order. Record acceptance, the next component to handle it, and the observed outcome. If only the first two events are available, leave the outcome pending.
