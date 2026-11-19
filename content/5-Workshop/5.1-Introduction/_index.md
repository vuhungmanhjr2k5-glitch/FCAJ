---
title: "Introduction and Architecture"
date: "2026-10-03"
weight: 1
chapter: false
pre: " <b> 5.1 </b> "
---

<!-- workshop-split:v1 -->

## Overview

In an e-commerce application, placing an order triggers several downstream tasks: recording the order, sending a confirmation email, and notifying administrators. If all of these are handled within the same request (synchronously), the system slows down easily as order volume grows, and a failure in a secondary service can cause lost orders or duplicate emails.

This workshop builds an order-processing flow that is **event-driven, fault-tolerant, and observable** on AWS, then runs measurements to compare it against a synchronous approach.

<!-- workshop-detail:v1:begin -->
## The problem to solve

A synchronous API may wait for database writes and email submission before responding. A slow or failing task can delay the response or return an error. If the visitor retries, the application must distinguish a new order from another attempt to submit the same order.

An asynchronous design separates acceptance from completion. The API places a request in a queue, while downstream components continue processing it. This requires clear order states, duplicate controls and observable failures.

## Design principles

- **Track by order ID:** correlate requests, database records and executions using the same identifier.
- **Expect redelivery:** SQS Standard does not guarantee that a message is delivered only once.
- **Persist processing state:** acceptance is not completion.
- **Use multiple evidence sources:** combine logs, DynamoDB, execution history and received email.

The four subsections explain the intended deliverables, architecture, evaluation approach and time/cost plan. Understand the design before deploying resources. This introductory section does not require additional Console screenshots or new resources.
<!-- workshop-detail:v1:end -->

## Workshop steps

Open a page below or use the sidebar. Follow the steps in order.

- [5.1.1 What you will do](step-01/)
- [5.1.2 Architecture](step-02/)
- [5.1.3 Evaluation metrics](step-03/)
- [5.1.4 Duration and cost](step-04/)
