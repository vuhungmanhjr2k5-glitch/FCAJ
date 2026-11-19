---
title: "Metrics and alarm configuration"
weight: 2
chapter: false
pre: " <b> 5.6.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

| Alarm | Namespace / metric | Dimension | Statistic |
|---|---|---|---|
| order-monitoring-evidence-sfn-failed | AWS/States / ExecutionsFailed | StateMachineArn | Sum |
| order-monitoring-evidence-lambda-errors | AWS/Lambda / Errors | FunctionName | Sum |
| order-monitoring-evidence-dlq | AWS/SQS / ApproximateNumberOfMessagesVisible | DLQ QueueName | Maximum |

All three use threshold **> 0**, period **60 seconds**, EvaluationPeriods **1**, DatapointsToAlarm **1** and TreatMissingData **notBreaching**. AlarmActions and OKActions reference the application's SNS topic.

Sum counts failures within a period; Maximum detects visible DLQ messages. SQS metrics are approximate. A 60-second period does not guarantee email delivery within 60 seconds. CloudWatch normally notifies on state transitions, rather than for every failure while ALARM persists.

## Match dimensions to the signal

A metric name identifies a quantity. `AWS/SQS` publishes it for many queues, so the DLQ alarm needs the destination `QueueName`. Selecting the source queue would count ordinary pending orders. The Lambda signal uses the physical function name. Step Functions uses the complete state machine ARN. A valid alarm pointed at the wrong dimension set may receive no relevant datapoints.

**Sum** adds execution or invocation errors within a minute. **Maximum** takes the highest observed visible-message count in that period. One DLQ message present across two periods can hold the alarm in `ALARM` for both and generate only the initial transition notification. Two datapoints of one may describe the same message.

## Read the template for your own account

The PowerShell block writes a local CloudFormation template. Section 5.6.3 supplies live resource identifiers. All three alarms use the application topic for both alarm and recovery actions. Each dashboard widget sets the metric Region explicitly. A dashboard named from its monitoring stack makes it straightforward to find after deployment.

Locate four resources in the saved file: three alarms and one dashboard. Compare every namespace and dimension with the opening table. Record any changes to period or missing-data treatment before experimenting. Those settings determine how to interpret the resulting notifications.

### Interpret a hypothetical evaluation period

Suppose the maximum DLQ visible count is zero, one and one across three periods. With a threshold of zero and the greater-than operator, the second period breaches. If the alarm remains in alarm, the third period does not cause a second transition into alarm. Another faulty message can require handling even if the operator receives only one email. This makes the initial alarm state part of the experiment record.

For Lambda and Step Functions, match the metric to the kind of incident. One malformed message can generate several failed invocations through redelivery. Several failed SES tasks can belong to one execution that increments ExecutionsFailed only on terminal failure. A value of one in the two namespaces counts different events.

### Save the monitoring template for 5.6.3

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

The four widgets use the same Region.Each metric row identifies the namespace, metric and dimension. Run both blocks in the same PowerShell window.

```powershell
$dashboardJson = @'
{
  "widgets": [
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Maximum",
        "metrics": [
              ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "${SourceQueueName}", {"label": "Source queue"}],
              ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "${DlqName}", {"label": "DLQ"}]
        ],
        "period": 60,
        "title": "SQS: visible messages",
        "view": "timeSeries"
      },
      "y": 0,
      "type": "metric",
      "x": 0,
      "height": 6
    },
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Maximum",
        "metrics": [
              ["AWS/SQS", "ApproximateAgeOfOldestMessage", "QueueName", "${SourceQueueName}"]
        ],
        "period": 60,
        "title": "SQS: oldest source message (seconds)",
        "view": "timeSeries"
      },
      "y": 0,
      "type": "metric",
      "x": 12,
      "height": 6
    },
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Sum",
        "metrics": [
              ["AWS/States", "ExecutionsStarted", "StateMachineArn", "${MachineArn}"],
              ["AWS/States", "ExecutionsSucceeded", "StateMachineArn", "${MachineArn}"],
              ["AWS/States", "ExecutionsFailed", "StateMachineArn", "${MachineArn}"]
        ],
        "period": 60,
        "title": "Step Functions: executions",
        "view": "timeSeries"
      },
      "y": 6,
      "type": "metric",
      "x": 0,
      "height": 6
    },
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Sum",
        "metrics": [
              ["AWS/Lambda", "Invocations", "FunctionName", "${WorkerName}"],
              ["AWS/Lambda", "Errors", "FunctionName", "${WorkerName}"]
        ],
        "period": 60,
        "title": "Lambda: invocations and errors",
        "view": "timeSeries"
      },
      "y": 6,
      "type": "metric",
      "x": 12,
      "height": 6
    }
  ]
}
'@
$null = $dashboardJson | ConvertFrom-Json -ErrorAction Stop
```

The second block defines the three alarms, places the dashboard JSON under Fn::Sub and writes the file. The resource placeholders remain literal until CloudFormation resolves them at deployment.

```powershell
if ([string]::IsNullOrWhiteSpace($dashboardJson)) { throw 'Run the dashboard block above first.' }
$definitionJson = @'
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Order workshop monitoring for a selected application stack",
  "Parameters": {
    "DlqName": {
      "Type": "String"
    },
    "MachineArn": {
      "Type": "String"
    },
    "SourceQueueName": {
      "Type": "String"
    },
    "SourceStack": {
      "Type": "String"
    },
    "TopicArn": {
      "Type": "String"
    },
    "WorkerName": {
      "Type": "String"
    }
  },
  "Resources": {
    "OrderDashboard": {
      "Properties": {
        "DashboardName": {
          "Fn::Sub": "${AWS::StackName}-dashboard"
        },
        "DashboardBody": {
          "Fn::Sub": ""
        }
      },
      "Type": "AWS::CloudWatch::Dashboard"
    },
    "SfnFailedAlarm": {
      "Properties": {
        "AlarmName": {
          "Fn::Sub": "${AWS::StackName}-sfn-failed"
        },
        "ComparisonOperator": "GreaterThanThreshold",
        "OKActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "AlarmActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "Dimensions": [
          {
            "Value": {
              "Ref": "MachineArn"
            },
            "Name": "StateMachineArn"
          }
        ],
        "EvaluationPeriods": 1,
        "Namespace": "AWS/States",
        "AlarmDescription": {
          "Fn::Sub": "Monitoring ${SourceStack}"
        },
        "TreatMissingData": "notBreaching",
        "MetricName": "ExecutionsFailed",
        "Statistic": "Sum",
        "Threshold": 0,
        "DatapointsToAlarm": 1,
        "Period": 60
      },
      "Type": "AWS::CloudWatch::Alarm"
    },
    "DlqAlarm": {
      "Properties": {
        "AlarmName": {
          "Fn::Sub": "${AWS::StackName}-dlq"
        },
        "ComparisonOperator": "GreaterThanThreshold",
        "OKActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "AlarmActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "Dimensions": [
          {
            "Value": {
              "Ref": "DlqName"
            },
            "Name": "QueueName"
          }
        ],
        "EvaluationPeriods": 1,
        "Namespace": "AWS/SQS",
        "AlarmDescription": {
          "Fn::Sub": "Monitoring ${SourceStack}"
        },
        "TreatMissingData": "notBreaching",
        "MetricName": "ApproximateNumberOfMessagesVisible",
        "Statistic": "Maximum",
        "Threshold": 0,
        "DatapointsToAlarm": 1,
        "Period": 60
      },
      "Type": "AWS::CloudWatch::Alarm"
    },
    "LambdaErrorAlarm": {
      "Properties": {
        "AlarmName": {
          "Fn::Sub": "${AWS::StackName}-lambda-errors"
        },
        "ComparisonOperator": "GreaterThanThreshold",
        "OKActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "AlarmActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "Dimensions": [
          {
            "Value": {
              "Ref": "WorkerName"
            },
            "Name": "FunctionName"
          }
        ],
        "EvaluationPeriods": 1,
        "Namespace": "AWS/Lambda",
        "AlarmDescription": {
          "Fn::Sub": "Monitoring ${SourceStack}"
        },
        "TreatMissingData": "notBreaching",
        "MetricName": "Errors",
        "Statistic": "Sum",
        "Threshold": 0,
        "DatapointsToAlarm": 1,
        "Period": 60
      },
      "Type": "AWS::CloudWatch::Alarm"
    }
  },
  "Outputs": {
    "DashboardName": {
      "Value": {
        "Ref": "OrderDashboard"
      }
    },
    "SourceStack": {
      "Value": {
        "Ref": "SourceStack"
      }
    }
  }
}
'@
$definition = $definitionJson | ConvertFrom-Json -ErrorAction Stop
$definition.Resources.OrderDashboard.Properties.DashboardBody.'Fn::Sub' = $dashboardJson
$content = $definition | ConvertTo-Json -Depth 30
$path = Join-Path $PWD.Path 'monitoring.template.json'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

The original environment-specific template is retained as a reference.

[monitoring.template.json](/downloads/5.6-Monitoring/evidence-20261114-043035-985e31/monitoring.template.json)
