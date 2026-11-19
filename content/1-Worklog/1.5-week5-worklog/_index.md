---
title: "Week 5 Worklog"
date: 2026-09-07
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

**Internship period:** 07/09/2026 - 12/09/2026

### Week 5 Objectives:

- Understand how event producers can be decoupled from processing components.
- Define event structures and practice routing with Amazon EventBridge.

### Tasks Completed:

| Date | Task | Reference Material |
|------|------|--------------------|
| 07/09/2026 | Compare direct invocation with event publication and identify producer and consumer responsibilities. | [Event-driven architecture](https://aws.amazon.com/event-driven-architecture/) |
| 08/09/2026 | Design order events containing eventId, orderId, eventType, schemaVersion, and occurrence time. | Draft event schema |
| 09/09/2026 | Create an event bus and publish test events to EventBridge. | [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) |
| 11/09/2026 | Configure event-type rules and verify delivery to a test target. | [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) |
| 12/09/2026 | Test events with missing fields, incorrect types, and unmatched rules; review the required payload data. | Event test notes |

### Week 5 Achievements:

- Defined event conventions for consistent communication between components.
- Practiced event publication and routing and identified cases that do not produce the expected processing behavior.