---
title: "Monitoring results and notifications"
weight: 5
chapter: false
pre: " <b> 5.6.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

The test message contains run ID **mon56-20261114-040056-316515**. Dashboard/alarm times are UTC; the inbox and SQS timestamps shown use UTC+07:00. For example, November 13, 2026, 21:03:44 UTC equals November 14, 2026, 04:03:44 UTC+07:00.

**Supplementary dashboard.** All four widgets and legends are visible; the DLQ already shows a value of 1 before the test. ExecutionsFailed is 0 in the dashboard capture; a separate alarm-detail capture records 1. The dashboard snapshot does not establish the final outcome of all executions.

![Supplementary dashboard](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h5.png)

**Step Functions.** order-monitoring-evidence-sfn-failed is ALARM and ExecutionsFailed reaches 1.

[Step Functions alarm](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h6.png)

**Lambda.** order-monitoring-evidence-lambda-errors is ALARM and Errors reaches 1. The metric screenshot alone does not identify the cause or MessageId of the failed invocation.

[Lambda Errors alarm](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h7.png)

**DLQ.** order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU contains not-json-mon56-20261114-040056-316515. An older message is also visible. Receive count is not treated as an exact Lambda attempt count because console receives can also affect it.

[Test message in the DLQ](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h8.png)

**Step Functions ALARM email.** The email records datapoint 1 at November 13, 2026, 21:02 UTC and transition to ALARM at 21:03:44 UTC. This establishes receipt of a CloudWatch notification. The screenshots lack the corresponding execution failure timestamp needed to calculate detection latency.

[Step Functions ALARM email](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h9.png)

**Lambda ALARM email.** The CloudWatch email reports an Errors datapoint of 1 at November 13, 2026, 21:01:00 UTC, exceeding the threshold of 0. Alarm order-monitoring-evidence-lambda-errors transitions from OK to ALARM at 21:02:21 UTC, corresponding to November 14, 2026, 04:02:21 UTC+07:00. The inbox screenshot establishes receipt of the ALARM notification, consistent with the CloudWatch alarm state. The datapoint timestamp is not the exact Lambda failure time and is not used to calculate detection or email-delivery latency.

[Lambda ALARM email](/images/5-Workshop/5.6-Monitoring/lambda-alarm-20261114-044826-a961f7/h10.png)

**DLQ ALARM email.** It records datapoint 1 at November 13, 2026, 20:59 UTC and ALARM at 21:01:06 UTC. That datapoint predates the newly sent message shown in the DLQ. Interpret it as notification of existing backlog, not a transition caused specifically by the new message.

[DLQ backlog ALARM email](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h11.png)

## Reconstruct the sequence from recorded times

Sort the events in UTC first. The DLQ transition at 21:01:06 UTC relates to a datapoint preceding the new malformed message. The Lambda transition at 21:02:21 UTC indicates Errors crossed its threshold. The Step Functions transition at 21:03:44 UTC indicates a period with one failed execution. This orders the alarms. Linking an invocation to an individual message still requires application evidence.

For example, the DLQ body containing `mon56-20261114-040056-316515` connects the run directly to its destination. The Errors graph aggregates events. A rise around the same time points to a useful log window but carries no message ID. Keep both timestamps if a dashboard view and an alarm detail view were captured at different times.

## Record your own observations

| Field | Source |
|---|---|
| Run ID and malformed marker | The run directory's `run.json` |
| DLQ baseline | `dlq-before.json` |
| Alarm and before/after states | Baseline, alarm history and received email |
| First workflow failure | Execution for the `-ses-fail` order |
| Missing milestone | Name the unavailable source, such as mailbox arrival time |

A metric timestamp identifies a statistical period. Computing detection latency needs both the actual failure and the alarm transition on the same clock basis. Measuring mail delivery needs a corresponding receipt. Limit conclusions from the recorded run to the milestones available.

### Read an alarm email from top to bottom

Start with the alarm name and Region to identify the deployment. Read the previous and new states, then the transition reason. The reason describes the datapoint that crossed the threshold and its evaluation period. Finally record the state change time. A mailbox can display local time, so retain the offset when comparing it with CloudWatch UTC.

The recorded Lambda and Step Functions transitions were 83 seconds apart, from 21:02:21 to 21:03:44 UTC. That subtraction describes the spacing between **two alarm transitions** from different services. It cannot measure how long Lambda took to detect its own fault or how long an email took to arrive. Two accurate timestamps can answer the wrong question if they describe the wrong milestones.

A reference to the matching alarm detail or existing figure, plus the alarm name and time, is sufficient for a reader to follow the timeline without opening every illustration.
