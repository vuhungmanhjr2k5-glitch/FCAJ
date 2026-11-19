---
title: "Introduction and Architecture"
date: "2026-10-03"
weight: 1
chapter: false
pre: " <b> 5.1 </b> "
---
## Overview

In an e-commerce application, placing an order triggers several downstream tasks: recording the order, sending a confirmation email, and notifying administrators. If all of these are handled within the same request (synchronously), the system slows down easily as order volume grows, and a failure in a secondary service can cause lost orders or duplicate emails.

This workshop builds an order-processing flow that is **event-driven, fault-tolerant, and observable** on AWS, then runs measurements to compare it against a synchronous approach.

## What you will do

- Decouple order intake from order processing using **Amazon SQS** and a dead-letter queue (**DLQ**)
- Prevent duplicate processing with **Amazon DynamoDB** (idempotency)
- Orchestrate the email and notification workflow with **AWS Step Functions**, **Amazon SES**, and **Amazon SNS**
- Monitor the system and receive alerts with **Amazon CloudWatch**
- Simulate failures and load to evaluate the effectiveness of the architecture

## Architecture

{{< figure src="/images/5-Workshop/5.1-Introduction/architecture.png" alt="Overall architecture" >}}

## Evaluation metrics

- Response time of the order API (average, p95)
- Rate of orders lost when failures occur
- Number of duplicate emails sent
- Time from when a failure occurs until an administrator receives an alert

## Duration and cost

Approximately 2–3 hours. Most services fall within the free tier or incur very low costs at test scale. Be sure to perform the cleanup step at the end to avoid incurring charges.