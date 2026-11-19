---
title: "Evaluation metrics"
weight: 3
chapter: false
pre: " <b> 5.1.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

- Response time of the order API (average, p95)
- Rate of orders lost when failures occur
- Number of duplicate emails sent
- Time from when a failure occurs until an administrator receives an alert

<!-- workshop-detail:v1:begin -->
## Separate the timing measurements

API response time runs from request submission to response receipt. In an asynchronous flow, that response may only confirm acceptance. Completion time ends when the order reaches its terminal state. Email receipt also includes delivery after SES accepts the message.

| Metric | Meaning | Evidence source |
|---|---|---|
| Mean API latency | Average response time within the run | Request-generation tool |
| API P95 | A response-time percentile representing the slower requests | The same tool and percentile method across runs |
| DONE / FAILED / processing | Order states at the observation time | DynamoDB and executions |
| Email tasks / received messages | Different counts that require separate checks | Execution history / recipient inbox |
| Alert delay | Time between a defined failure point and a defined alert point | Execution, alarm history or email |

## Record comparable experiments

Record the run ID, request count, concurrency, pauses, start time and reconciliation time. Compare implementations under equivalent conditions and use the same P95 calculation. Small samples characterize the tested run, not large-scale capacity.

A record not found at one observation does not establish permanent loss. Correlate the API response, queue and logs, then check again after an appropriate wait. An alarm entering ALARM and an email arriving in the inbox are different timestamps.

Suggested recording fields: run ID, requests, successful API responses, DONE, FAILED, unfinished, mean API latency, API P95 and observation time. No load run or additional screenshot is required in this introductory section.
<!-- workshop-detail:v1:end -->

## Calculate a percentile for a small run

Take 20 HTTP responses and sort their durations from shortest to longest. The nearest-rank P95 is observation number 19, counting from one, because the ceiling of `0.95 × 20` is 19. A very slow twentieth response may pull up the mean while leaving P95 unchanged. Record the number of measured responses beside the percentile. A network exception that produced no HTTP response belongs in the network error count.

## Choose the denominator before reporting completion

For a worked example, submit 20 distinct IDs and receive 20 acceptances. At the one-minute check the table contains 17 `DONE`, one `FAILED` and two `RECEIVED`. Completion at that instant is 17/20. Both pending orders are traceable. Recheck after the workflow has had time to retry, and keep a second timestamped row. These figures illustrate the calculation only.

For the duplicate test, five submissions of one ID represent five attempts and one purchase. Record requests, distinct IDs, executions and successful email tasks separately. A table containing a single item can otherwise conceal more than one email operation.
