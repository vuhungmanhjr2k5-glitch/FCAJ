---
title: "Conclusion"
weight: 9
chapter: false
pre: " <b> 5.7.9 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Both modes reached DONE for 20/20 orders in this run. Separate tests recorded duplicate handling, SES retries, SNS failure notifications and DLQ alerts. Findings are limited to the documented test conditions and runs.

## Choose the architecture using both timelines

For a customer waiting at checkout, the relevant interval begins when the client sends the request and ends on intake confirmation. In the recorded run both groups had 20 successful HTTP responses. Client-side means were 229.55 ms for sync and 109.28 ms for async. The async storefront can immediately present the order ID and acceptance status. Reporting completion at that point would get ahead of the workflow.

For support staff checking progress, the relevant observation is the final `DONE` or `FAILED` for that ID. Approximate mean completion was 122.75 ms for sync and 445.55 ms for async in the measured groups. Those times come from application fields. Their relationship to client latency depends on clock boundaries and how much work continues outside the request. Treating them as interchangeable leads to incorrect status messages.

## Link the conclusion to the experiment

| Experiment | Recorded observation | Scope |
|---|---|---|
| Small API run | 20/20 DONE for each mode at concurrency one | Async intake responded faster in the recorded conditions |
| Five sends of one body | One verified execution, one successful SES task, counter one | Duplicate handling worked for that ID and scenario |
| Rejected email | Error, retry and administrative path in history | A failure after acceptance can be traced |
| Malformed body | Matching DLQ body and an observed alarm | Failed input can be retained for investigation |
| Incident notification latency | Matching timestamp pair unavailable | The interval remains unmeasured |

Include submitted requests, distinct order IDs, the observation time and actual sample count with a comparison. Treat a change of recipient type or worker concurrency as a new run. Retain the original population as a reference.

## Hand over results before cleanup

Choose one successful and one failed order you actually submitted. For each, record the API path or queue source, final state, evidence inspected and any open question. For invalid input lacking an order ID, use the body marker and DLQ name. Together with the latency table, these records let another reader check the reasoning without opening your AWS account.

Before section 5.8 removes resources, save the prefixes, benchmark output and final observation time. This matters for logs and items belonging to the stack that will be removed. Billing needs its own check for newly accrued charges. A configured budget or a small value on a single billing view does not supply that measurement.

### Revisit the conclusion when the business requirement changes

A merchant who needs only “request received” while the customer is at checkout can evaluate the first screen using API latency and a traceable order ID. The customer can check the final state afterward. If the success screen promises that the email was submitted and processing was recorded, the measurement must extend to completion. The recorded table supplies two distinct intervals for that decision.

Another requirement might give the operator a notification deadline. Received emails establish that a notification channel worked on the observed run. The missing timestamp of the initiating fault still prevents a fault-to-notification measurement. Name which requirement was exercised and which needs an additional observation. A later run can use the same resources while recording the original error time and correlating alarm history with the corresponding inbox message.

### Write a result paragraph from a new run

“Run … in Singapore used … distinct orders per mode at concurrency …. API mean/P95 were … for sync and … for async. At … UTC the table contained … DONE, … FAILED and … pending. Order … demonstrated the failure path at state …. The … measure lacks source or timestamp ….” Fill the fields from observations and leave unresolved fields open. This places conditions, measurements and limitations together for a fair comparison with the recorded run.
