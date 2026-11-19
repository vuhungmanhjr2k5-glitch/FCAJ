---
title: "Assessment and limits"
weight: 6
chapter: false
pre: " <b> 5.6.6 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

| Area | Supported conclusion |
|---|---|
| Dashboard | Four metric groups are visible |
| Step Functions / Lambda alarms | Both are observed in ALARM |
| New test message in DLQ | Matching body and queue are visible |
| Step Functions / DLQ ALARM emails | Received; DLQ notification concerns backlog |
| Lambda ALARM email | ALARM notification received, matching the alarm name and Errors = 1 datapoint |
| Recovery and alert latency | Insufficient evidence to establish recovery or calculate duration |

Polling makes DLQ messages temporarily invisible and can reduce visible counts without resolution/deletion. OK with notBreaching does not independently establish application health. Retain run.json, sent.json, alarms-before.json and dlq-before.json for correlation; do not infer exact order totals or individual outcomes solely from metrics.

## Examine an apparent return to OK

Suppose an operator polls the DLQ to inspect a body. The visible count can fall during the visibility interval even though the message remains. Without a breaching point in the next period, the alarm may return to `OK`. To establish business recovery, check how the item was handled, the final order state and whether the message returned after the visibility timeout.

An interval without Lambda Errors can likewise occur with no traffic at all. Read Invocations and trace a valid order over the same interval. A workflow that is still retrying has not yet registered a terminal failed execution.

## Bound the conclusion in the report

Use three sentences. Start with a direct observation, such as a matching test body in the expected DLQ. State an explanation that still needs evidence, such as a possible link between a Lambda error datapoint and a parse failure. Finish with the observation that could distinguish it, such as the timestamped log entry and message ID. This format works with text-only evidence.

Leave detection latency blank when the initiating error was not timestamped. Mark recovery as untested when it was not observed. These limits identify the additional data needed before turning a lab observation into an operational commitment.

### Design a later recovery check

Choose the specific capability that should recover. For an SES identity issue, the objective could be a **new** order with a valid recipient reaching `DONE` after correction. For the DLQ, it could be a particular message investigated, corrected at the input and handled under the application's recovery process. Sending the same malformed text back to the source would encounter the same parse failure.

Identify a starting observation, intervention and checkable end state. For example, record the failed order, correct the test recipient in a new payload, submit a fresh ID and check the email task and table update. Label this as a check of new traffic after a correction. The older item still describes the original failure.

| Open question | Evidence needed in a later run |
|---|---|
| How long did detection take? | Timestamped original fault and corresponding alarm transition |
| How long did notification delivery take? | Same notification, transition time and inbox receipt time |
| Was the backlog resolved? | Message identifiers, final handling outcome and an observation after the visibility timeout |

This table is a measurement plan. Enter recovery results only after the corresponding run has taken place.
