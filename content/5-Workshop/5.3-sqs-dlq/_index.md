---
title: "Queues and DLQ"
weight: 3
chapter: false
pre: " <b> 5.3 </b> "
---

<!-- workshop-split:v1 -->

## Objective

Create a main queue that receives orders and a dead-letter queue (**DLQ**)
to keep messages that fail processing, instead of losing them.

Three concepts to remember:

- **Visibility timeout**: the time a message stays hidden after a consumer receives it. If the consumer does not delete the message within this time, the message becomes visible again so it can be retried.
- **maxReceiveCount**: the number of times a message can be received without being deleted before it is moved to the DLQ.
- **DLQ**: a queue that holds failed messages so they can be inspected and reprocessed.

## Workshop steps

Open a page below or use the sidebar. Follow the steps in order.

- [5.3.1 Create the template.yaml file](step-01/)
- [5.3.2 Deploy](step-02/)
- [5.3.3 Verify in the Console](step-03/)
- [5.3.4 Test a message falling into the DLQ](step-04/)
- [5.3.5 Verify the result](step-05/)
