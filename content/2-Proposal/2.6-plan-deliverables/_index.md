---
title: "Implementation Plan and Deliverables"
weight: 6
chapter: false
pre: " <b> 2.6. </b> "
---

## 1. Implementation Plan

Implementation will focus on the final four weeks, using the knowledge and experimental components prepared earlier.

| Period | Main Activities | Expected Output |
|--------|-----------------|-----------------|
| Week 9: 05/10–10/10/2026 | Finalize the architecture and integrate APIs, queues, consumers, and state storage. | Two runnable processing flows, architecture diagrams, and API specifications. |
| Week 10: 12/10–17/10/2026 | Refine idempotency, retries, DLQ handling, logs, and alarms; run failure tests. | Failure test records and recovery instructions. |
| Week 11: 19/10–24/10/2026 | Run workload scenarios, collect data, and compare the implementations. | Raw data, summary tables, and evaluation charts. |
| Week 12: 26/10–31/10/2026 | Review results and finalize documentation, evidence, and the report. | Source code, implementation guide, and final report. |

## 2. Expected Deliverables and Acceptance Criteria

### 2.1. Deliverables

- Source code for the synchronous and event-driven implementations.
- Architecture diagrams and processing-flow documentation.
- Deployment, configuration, and resource-cleanup instructions.
- Test data, tools, and experimental scenarios.
- Dashboards, log queries, and alarm configurations.
- Retry, DLQ, and idempotency test records.
- A comparison report supported by data and evidence.
- Vietnamese and English worklogs and summary documentation.

### 2.2. Acceptance Criteria

- Both implementations satisfy the agreed business scope.
- Order states and processing progress can be inspected by identifier.
- Failure behavior is tested through reproducible scenarios.
- Failed-message recovery is documented and exercised.
- Duplicate-processing controls are verified for the defined business effects.
- Comparisons distinguish response latency from completion time.
- Limitations, failed cases, and unresolved issues are explicitly recorded.

The project does not assume that event-driven processing is always superior. The expected outcome is an evidence-based understanding of when each approach is suitable and what trade-offs it introduces.
