---
title: "Week 2 Worklog"
date: 2026-08-17
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

**Internship period:** 17/08/2026 - 22/08/2026

### Week 2 Objectives:

- Understand request-response processing and how sequential operations affect response time.
- Design the API interface and shared order-state conventions.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 17/08/2026 | Study request-response processing, HTTP status codes, and request timeout scenarios. | [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) |
| 18/08/2026 | Design the create-order request structure, input validation rules, and error response format. | Draft API specification |
| 19/08/2026 | Define order identifiers and states for accepted, processing, completed, and failed orders. | State model notes |
| 21/08/2026 | Create a test endpoint in API Gateway and submit valid and invalid requests. | [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) |
| 22/08/2026 | Refine the synchronous sequence diagram and identify where processing timestamps should be recorded. | Draft processing diagram |

### Week 2 Achievements:

- Created the initial specification for order creation and status-query APIs.
- Identified the synchronous processing steps and the error scenarios requiring handling.