---
title: "Monitoring objective and architecture"
weight: 1
chapter: false
pre: " <b> 5.6.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Observe SQS, Lambda and Step Functions with CloudWatch metrics, three alarms, a dashboard and SNS email in **ap-southeast-1**. The supplementary **order-monitoring-evidence** stack monitors **order-orchestration-evidence** application resources.

## Observe an order and observe the system

Use a DynamoDB key and execution history to locate an individual order. CloudWatch answers a broader question about which component showed symptoms during a time window. A Lambda invocation failure can increment Errors without including an order ID in the datapoint. Keep the timestamp, function name and Region so you can return to the log.

The dashboard groups four signals. Visible messages describe the backlog available to consumers. Oldest-message age highlights prolonged waiting. Execution metrics show workflow starts and outcomes. Lambda invocation and error metrics show worker activity. Backlog rising along with age suggests that arrivals outpaced processing during the selected interval.

## Attach monitoring to your deployment

The runnable example uses the `order-system` application and a separate `order-monitoring` stack. Monitoring receives queue and function names, the state machine ARN and the SNS topic ARN from the application. Names ending in `evidence` in the illustrations belong to the recorded environment. Resolve your own resources so the dashboard follows the deployment that receives your test traffic.

An alarm can notify on a state transition, while the workflow can publish for an individual failed order. A CloudWatch notification identifies an alarm and threshold. A business notification carries an order ID. Identify the source before using email timestamps for measurement.

### Choose a first chart for a delay complaint

Suppose customers report a delay without an error alarm. Check oldest-message age and source depth. If both increase while executions succeed, completions may be lagging behind intake. A small source backlog with executions waiting for retries places the delay downstream. The configured alarms target particular errors, so a slow run can still require dashboard inspection.
