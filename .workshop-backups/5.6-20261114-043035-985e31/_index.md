---
title: "Monitoring"
weight: 6
chapter: false
pre: " <b> 5.6 </b> "
---
## Objective

A fault-tolerant system is only valuable if operators **know** when something
goes wrong. In this section we add:

- Three **CloudWatch Alarms**: messages in the DLQ, a failed execution, and Lambda errors.
- A **Dashboard** that shows the entire processing flow on a single screen.

Alerts are sent through the same SNS topic from section 5.5, so the email that
receives alerts is the admin email you confirmed earlier.

#### 1. Add the alarms to template.yaml

Insert the following block at the end of the `Resources` section, right before the `Outputs:` line
(keep the indentation of exactly 2 spaces):

```yaml
  DlqAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: order-dlq-has-messages
      AlarmDescription: There are messages in the DLQ
      Namespace: AWS/SQS
      MetricName: ApproximateNumberOfMessagesVisible
      Dimensions:
        - Name: QueueName
          Value: !GetAtt OrdersDLQ.QueueName
      Statistic: Maximum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 0
      ComparisonOperator: GreaterThanThreshold
      TreatMissingData: notBreaching
      AlarmActions:
        - !Ref OrderTopic

  SfnFailedAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: order-sfn-failed
      AlarmDescription: A Step Functions execution has failed
      Namespace: AWS/States
      MetricName: ExecutionsFailed
      Dimensions:
        - Name: StateMachineArn
          Value: !Ref OrderStateMachine
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 0
      ComparisonOperator: GreaterThanThreshold
      TreatMissingData: notBreaching
      AlarmActions:
        - !Ref OrderTopic

  LambdaErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: order-lambda-errors
      AlarmDescription: The ProcessOrderFn Lambda reported errors
      Namespace: AWS/Lambda
      MetricName: Errors
      Dimensions:
        - Name: FunctionName
          Value: !Ref ProcessOrderFn
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 0
      ComparisonOperator: GreaterThanThreshold
      TreatMissingData: notBreaching
      AlarmActions:
        - !Ref OrderTopic

  OrderDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: order-workshop
      DashboardBody: !Sub |
        {
          "widgets": [
            { "type": "metric", "x": 0, "y": 0, "width": 12, "height": 6,
              "properties": { "title": "SQS - messages waiting", "region": "${AWS::Region}",
                "view": "timeSeries", "stat": "Maximum", "period": 60,
                "metrics": [
                  ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "${OrdersQueue.QueueName}", {"label": "Main queue"}],
                  ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "${OrdersDLQ.QueueName}", {"label": "DLQ"}]
                ] } },
            { "type": "metric", "x": 12, "y": 0, "width": 12, "height": 6,
              "properties": { "title": "SQS - age of oldest message (seconds)", "region": "${AWS::Region}",
                "view": "timeSeries", "stat": "Maximum", "period":