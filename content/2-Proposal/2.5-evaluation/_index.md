---
title: "Experimental Methodology"
weight: 5
chapter: false
pre: " <b> 2.5. </b> "
---

## 1. Experimental Methodology

### 1.1. Comparison Principles

Both implementations will use the same business scope, input dataset, and definition of a completed order.

Resource configuration, concurrency limits, retry policies, source revisions, and execution conditions will be recorded for reproducibility.

Two timing measurements must be distinguished:

- **API response latency:** the time from the client sending a request to receiving its response.
- **Order completion time:** the time from the original request to the order reaching the defined terminal state.

A fast asynchronous acceptance response will not be treated as proof of faster business completion.

### 1.2. Experimental Scenarios

| Scenario | Evaluation Focus |
|----------|------------------|
| Steady load | Latency and throughput under a relatively stable arrival rate. |
| Burst load | Intake behavior, backlog growth, and queue-drain time. |
| Slow processing step | Effects on API responses and order completion time. |
| Transient failure | Retry behavior and completion after the failure is removed. |
| Persistent failure | Message isolation, alarms, and recovery through the DLQ. |
| Request or message resubmission | Control of duplicate business effects. |

### 1.3. Evaluation Metrics

- p50 and p95 API response latency.
- p50 and p95 completion time for completed orders.
- Unique orders completed per unit of time.
- Completed, failed, and unfinished order rates at the observation cutoff.
- Backlog size and queue-drain time.
- Recovery time after the failure cause is removed.
- Detected duplicate business effects.
- Experimental cost where sufficient allocation data is available.

Failed and unfinished orders will be reported separately rather than silently excluded from analysis.

Each scenario will be repeated. The report will document run count, duration, workload, and variation between runs.
