---
title: "Overview and Objectives"
weight: 1
chapter: false
pre: " <b> 2.1. </b> "
---

## 1. Project Title

**Building a Fault-Tolerant and Observable Event-Driven Order Processing Workflow on AWS and Comparing It with Synchronous Processing.**

## 2. Background and Problem Statement

In an e-commerce application, accepting an order can trigger several operations, including input validation, order recording, state updates, and notifications.

In a synchronous implementation, the client waits for the operations included in the request path to finish before receiving a result. Slow processing or dependency failures can increase response time and cause request timeouts.

Resubmitting a request after a timeout can also produce duplicate processing if appropriate controls are absent. However, a timeout does not necessarily mean that the order was never processed: some operations may have completed before the connection was interrupted.

An event-driven architecture separates order acceptance from downstream processing through events or messages. This can allow requests to be accepted while processing components are slow and enable retries after transient failures.

The approach also introduces challenges, including duplicate messages, delayed state visibility, queue backlogs, and tracing work across multiple components.

This project proposes two implementations with equivalent business scope and evaluates their differences through controlled experiments.

## 3. Project Objectives

### 3.1. Overall Objective

Build an event-driven order processing model on AWS with failure handling and state visibility, then compare it with synchronous processing using experimental data.

### 3.2. Specific Objectives

- Implement order acceptance and status-query APIs.
- Build a synchronous implementation as the comparison baseline.
- Build an asynchronous implementation using a queue and an independent processing component.
- Design retries, dead-letter handling, and failed-message recovery.
- Use idempotency to control the effects of repeated requests or messages.
- Correlate logs across components to follow individual orders.
- Create dashboards and alarms for failures, backlogs, and unsuccessful messages.
- Measure latency, completion throughput, success rate, and recovery behavior.
- Identify the suitable conditions, limitations, and trade-offs of each architecture.
