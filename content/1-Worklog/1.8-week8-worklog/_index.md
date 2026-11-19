---
title: "Week 8 Worklog"
date: 2026-09-28
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

**Internship period:** 28/09/2026 - 03/10/2026

### Week 8 Objectives:

- Define a comparison methodology using the same business scope for both architectures.
- Prepare input data, workload scenarios, and result collection procedures.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 28/09/2026 | Define evaluation metrics: API response latency, order completion time, completion throughput, and success rate. | Experiment evaluation outline |
| 29/09/2026 | Prepare shared input data and define unique-order counting so retries are not counted as new orders. | Test dataset |
| 30/09/2026 | Design steady-load, burst-load, and slow-processing scenarios; specify run duration and repetitions. | Load test plan |
| 02/10/2026 | Build a request generator and record timestamps by order identifier; verify measurement consistency. | Test tool source code |
| 03/10/2026 | Run a small pilot test and review resource configuration, initialization behavior, and the time required to drain queues. | Pilot test notes |

### Week 8 Achievements:

- Prepared a measurement method that distinguishes request acceptance from business completion.
- Defined test scenarios and data collection rules to support reproducible comparisons.