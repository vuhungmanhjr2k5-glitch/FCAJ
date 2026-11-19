---
title: "Week 4 Worklog"
date: 2026-08-31
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

**Internship period:** 31/08/2026 - 05/09/2026

### Week 4 Objectives:

- Implement synchronous processing within the defined business scope.
- Prepare a baseline that supports repeatable testing and measurement.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 31/08/2026 | Connect API Gateway to the order processing Lambda function and verify invocation permissions. | [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) |
| 01/09/2026 | Implement input validation, the experimental business operations, and result updates as a synchronous sequence. | Project business specification |
| 02/09/2026 | Add error responses and check consistency between API results and stored order states. | API test notes |
| 04/09/2026 | Simulate a slow processing step to observe waiting time and timeout behavior. | [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) |
| 05/09/2026 | Record configuration, source revision, and execution steps so the synchronous baseline can be reproduced. | Experiment configuration notes |

### Week 4 Achievements:

- Built a synchronous baseline for comparison with event-driven processing.
- Established checks for responses, order states, and the effects of slow processing steps.