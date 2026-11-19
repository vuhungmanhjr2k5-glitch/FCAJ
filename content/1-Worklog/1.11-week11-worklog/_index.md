---
title: "Week 11 Worklog"
date: 2026-10-19
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

**Internship period:** 19/10/2026 - 24/10/2026

### Week 11 Objectives:

- Collect data for both approaches using the same experimental scenarios.
- Analyze performance and failure behavior using measured results.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 19/10/2026 | Freeze the source revision, resource settings, input data, and workload limits; document configuration differences between the approaches. | Experiment configuration record |
| 20/10/2026 | Run steady-load scenarios for synchronous and event-driven processing and retain raw data for each run. | Test tools and scenarios |
| 21/10/2026 | Run burst-load and slow-processing scenarios; monitor errors, backlog, and queue-drain time. | Experiment monitoring dashboard |
| 23/10/2026 | Calculate p50 and p95 response and completion times; measure throughput using unique orders and account for failed and unfinished orders. | Measurement data and results worksheet |
| 24/10/2026 | Compare repeated runs and analyze trade-offs involving latency, burst handling, and operational complexity. | Draft results analysis |

### Week 11 Achievements:

- Created a comparison table separating API response latency from order completion time.
- Identified factors affecting the measurements, including resource settings, function initialization, retries, and queue waiting time.
- Measurements to add: [Workload; run count; completed, failed, and unfinished orders; p50/p95; throughput; and recovery time for each approach.]