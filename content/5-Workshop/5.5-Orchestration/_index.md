---
title: "Orchestration and Notifications"
weight: 5
chapter: false
pre: " <b> 5.5 </b> "
---

<!-- workshop-split:v1 -->

## Objective

Instead of letting a single Lambda function do everything, we use **AWS Step Functions** to
orchestrate the order-processing workflow. The email task defines **Retry** and
**Catch** behavior, the state of every order is recorded, and administrators
are notified immediately through **Amazon SNS** when an order fails.

Workflow:

1. Lambda receives a message from SQS, records it in DynamoDB (`RECEIVED`), and starts the state machine.
2. The state machine sends a confirmation email to the customer using **Amazon SES** (up to 3 retries).
3. On success: update the status to `DONE`.
4. On failure: update the status to `FAILED`, send an alert through SNS, and end in a failed state.

![Workflow diagram](/images/5-Workshop/5.5-Orchestration/workflow.png)

## Workshop steps

Open a page below or use the sidebar. Follow the steps in order.

- [5.5.1 Replace the entire template.yaml](step-01/)
- [5.5.2 Replace the entire src/index.mjs](step-02/)
- [5.5.3 Deploy](step-03/)
- [5.5.4 Testing](step-04/)
- [5.5.5 Verify the results](step-05/)
- [5.5.6 Troubleshooting](step-06/)
