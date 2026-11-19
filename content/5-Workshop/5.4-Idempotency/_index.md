---
title: "Preventing Duplicate Processing"
weight: 4
chapter: false
pre: " <b> 5.4 </b> "
---

<!-- workshop-split:v1 -->

## Objective

An SQS queue guarantees **at-least-once** delivery, which means the same order
may occasionally be delivered twice. If this is not handled, customers will
receive duplicate emails. We use **DynamoDB** as a "ledger": each `orderId` is
processed only once. This technique is called **idempotency**.

## Workshop steps

Open a page below or use the sidebar. Follow the steps in order.

- [5.4.1 Update template.yaml](step-01/)
- [5.4.2 Write the Lambda code](step-02/)
- [5.4.3 Redeploy](step-03/)
- [5.4.4 Test](step-04/)
- [5.4.5 Verify the result](step-05/)
- [5.4.6 Troubleshooting](step-06/)
