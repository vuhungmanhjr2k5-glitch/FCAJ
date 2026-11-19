---
title: "Week 3 Worklog"
date: 2026-08-24
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

**Internship period:** 24/08/2026 - 29/08/2026

### Week 3 Objectives:

- Practice implementing processing logic with AWS Lambda.
- Design order data and store processing states in Amazon DynamoDB.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 24/08/2026 | Study Lambda handlers, execution roles, timeouts, and memory configuration. | [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) |
| 25/08/2026 | Implement input validation and identifier generation for test orders. | [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) |
| 26/08/2026 | Design a DynamoDB table around the requirement to retrieve orders by identifier. | [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) |
| 28/08/2026 | Integrate Lambda with DynamoDB to record orders and update their processing states. | [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) |
| 29/08/2026 | Test read, write, and update operations; review access permissions and responses for missing orders. | Functional test notes |

### Week 3 Achievements:

- Implemented the basic functions for accepting and retrieving orders.
- Developed a data model for processing states and timestamps needed for workflow monitoring.