---
title: "Conclusion"
weight: 8
chapter: false
pre: " <b> 5.6.8 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

The evidence records a dashboard, Step Functions/Lambda alarms, the test message in the DLQ, and received ALARM emails for all three alarms: Step Functions, Lambda and DLQ. The DLQ notification reflects backlog that predates the test. The screenshots do not establish post-failure recovery or alert latency measured from the actual failure time.

## Answer three questions for the operator

**Which layer reported trouble?** Select the relevant dashboard window. Compare Lambda and Step Functions trends to distinguish a failure reading a message from one occurring after workflow startup. Check both depth and age to see whether this was a single bad input or an interval of waiting orders.

**Which order is affected?** Use the metric window to locate a log or execution and recover the order ID or body marker. Malformed JSON can fail before any business key is read. The destination queue and its body can establish the link directly.

**Did the operator receive a notification?** Inspect alarm history for a transition and search the inbox for the matching alarm name. If the application workflow also publishes to the same topic, read the subject and content. An order-specific email and a datapoint notification can arise from one incident through two separate publication paths.

## Hand monitoring over to the API experiment

Record both stack names, Region and the three alarms. Note any persistent alarm, existing DLQ item or conclusion missing a timestamp. Section 5.7 reuses the queue, table and workflow, so an existing issue here can affect later measurements. Start the comparison with a known baseline. A table of values and a short interpretation are sufficient for this handover.
