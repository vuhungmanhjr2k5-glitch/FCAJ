---
title: "Fault and Load Experiments"
weight: 7
chapter: false
pre: " <b> 5.7 </b> "
---

<!-- workshop-split:v1 -->

## Objective

Compare two ways of processing orders under the same conditions:

| | Synchronous (baseline) | Asynchronous (proposed) |
|---|---|---|
| Flow | The API writes to DynamoDB and then sends via SES within the request | The API only puts the order into SQS; Lambda and Step Functions process it later |
| Duplicate protection | No | Yes (DynamoDB) |
| Retry, DLQ, alerts | No | Yes |

Metrics: API response time (average, p95), completed orders, failed or lost
orders, processing completion time, extra emails when duplicates are sent, and
the time until an alert is raised on failure.

>**SES sandbox limit:** at most 1 email/second. Emails sent to
>`success@simulator.amazonses.com` (the SES mailbox simulator) do not count
>toward the 200 emails/day quota but are still limited to 1 email/second. That
>is why the test loads are 20 and 50 orders. This is also a result worth
>recording: the bottleneck is the email-sending service, not the architecture.

## Workshop steps

Open a page below or use the sidebar. Follow the steps in order.

- [5.7.1 Update template.yaml](step-01/)
- [5.7.2 Write the code](step-02/)
- [5.7.3 Deploy](step-03/)
- [5.7.4 Prepare the measurement tools](step-04/)
- [5.7.5 Experiment 1: latency and load](step-05/)
- [5.7.6 Experiment 2: duplicate orders](step-06/)
- [5.7.7 Experiment 3: failures and alerts](step-07/)
- [5.7.9 Conclusion](step-09/)
