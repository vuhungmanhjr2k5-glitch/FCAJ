---
title: "Monitoring"
weight: 6
chapter: false
pre: " <b> 5.6 </b> "
---

## Monitoring objective and architecture

Observe SQS, Lambda and Step Functions with CloudWatch metrics, three alarms, a dashboard and SNS email in **ap-southeast-1**. The supplementary **order-monitoring-evidence** stack monitors **order-orchestration-evidence** application resources.

H1–H4 belong to the earlier **order-system** deployment. H5–H11 document supplementary monitoring. The two groups are not presented as one before/after run.

## 1. Metrics and alarm configuration

| Alarm | Namespace / metric | Dimension | Statistic |
|---|---|---|---|
| order-monitoring-evidence-sfn-failed | AWS/States / ExecutionsFailed | StateMachineArn | Sum |
| order-monitoring-evidence-lambda-errors | AWS/Lambda / Errors | FunctionName | Sum |
| order-monitoring-evidence-dlq | AWS/SQS / ApproximateNumberOfMessagesVisible | DLQ QueueName | Maximum |

All three use threshold **> 0**, period **60 seconds**, EvaluationPeriods **1**, DatapointsToAlarm **1** and TreatMissingData **notBreaching**. AlarmActions and OKActions reference the application's SNS topic.

Sum counts failures within a period; Maximum detects visible DLQ messages. SQS metrics are approximate. A 60-second period does not guarantee email delivery within 60 seconds. CloudWatch normally notifies on state transitions, rather than for every failure while ALARM persists.

### Deployment template

The following file is copied from the JSON generated on the lab computer. Resource identifiers are specific to this environment and need adjustment elsewhere.

[monitoring.template.json](/downloads/5.6-Monitoring/evidence-20261114-043035-985e31/monitoring.template.json)

```json
{
    "Outputs":  {
                    "DashboardName":  {
                                          "Value":  "order-monitoring-evidence-dashboard"
                                      },
                    "SourceStack":  {
                                        "Value":  "order-orchestration-evidence"
                                    }
                },
    "Description":  "Workshop 5.6 monitoring for order-orchestration-evidence",
    "AWSTemplateFormatVersion":  "2010-09-09",
    "Resources":  {
                      "OrderDashboard":  {
                                             "Properties":  {
                                                                "DashboardName":  "order-monitoring-evidence-dashboard",
                                                                "DashboardBody":  "{\"widgets\":[{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Maximum\",\"metrics\":[[\"AWS/SQS\",\"ApproximateNumberOfMessagesVisible\",\"QueueName\",\"order-orchestration-evidence-OrdersQueue-Q4yad2S16mVq\",{\"label\":\"Source queue\"}],[\"AWS/SQS\",\"ApproximateNumberOfMessagesVisible\",\"QueueName\",\"order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU\",{\"label\":\"DLQ\"}]],\"period\":60,\"title\":\"SQS: visible messages\",\"view\":\"timeSeries\"},\"y\":0,\"type\":\"metric\",\"x\":0,\"height\":6},{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Maximum\",\"metrics\":[[\"AWS/SQS\",\"ApproximateAgeOfOldestMessage\",\"QueueName\",\"order-orchestration-evidence-OrdersQueue-Q4yad2S16mVq\"]],\"period\":60,\"title\":\"SQS: oldest source message (seconds)\",\"view\":\"timeSeries\"},\"y\":0,\"type\":\"metric\",\"x\":12,\"height\":6},{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Sum\",\"metrics\":[[\"AWS/States\",\"ExecutionsStarted\",\"StateMachineArn\",\"arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly\"],[\"AWS/States\",\"ExecutionsSucceeded\",\"StateMachineArn\",\"arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly\"],[\"AWS/States\",\"ExecutionsFailed\",\"StateMachineArn\",\"arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly\"]],\"period\":60,\"title\":\"Step Functions: executions\",\"view\":\"timeSeries\"},\"y\":6,\"type\":\"metric\",\"x\":0,\"height\":6},{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Sum\",\"metrics\":[[\"AWS/Lambda\",\"Invocations\",\"FunctionName\",\"order-orchestration-evidence-ProcessOrderFn-A3d416bkFPjD\"],[\"AWS/Lambda\",\"Errors\",\"FunctionName\",\"order-orchestration-evidence-ProcessOrderFn-A3d416bkFPjD\"]],\"period\":60,\"title\":\"Lambda: invocations and errors\",\"view\":\"timeSeries\"},\"y\":6,\"type\":\"metric\",\"x\":12,\"height\":6}]}"
                                                            },
                                             "Type":  "AWS::CloudWatch::Dashboard"
                                         },
                      "SfnFailedAlarm":  {
                                             "Properties":  {
                                                                "AlarmName":  "order-monitoring-evidence-sfn-failed",
                                                                "ComparisonOperator":  "GreaterThanThreshold",
                                                                "OKActions":  [
                                                                                  "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                              ],
                                                                "AlarmActions":  [
                                                                                     "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                                 ],
                                                                "Dimensions":  [
                                                                                   {
                                                                                       "Value":  "arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly",
                                                                                       "Name":  "StateMachineArn"
                                                                                   }
                                                                               ],
                                                                "EvaluationPeriods":  1,
                                                                "Namespace":  "AWS/States",
                                                                "AlarmDescription":  "Workshop 5.6; source stack: order-orchestration-evidence",
                                                                "TreatMissingData":  "notBreaching",
                                                                "MetricName":  "ExecutionsFailed",
                                                                "Statistic":  "Sum",
                                                                "Threshold":  0,
                                                                "DatapointsToAlarm":  1,
                                                                "Period":  60
                                                            },
                                             "Type":  "AWS::CloudWatch::Alarm"
                                         },
                      "DlqAlarm":  {
                                       "Properties":  {
                                                          "AlarmName":  "order-monitoring-evidence-dlq",
                                                          "ComparisonOperator":  "GreaterThanThreshold",
                                                          "OKActions":  [
                                                                            "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                        ],
                                                          "AlarmActions":  [
                                                                               "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                           ],
                                                          "Dimensions":  [
                                                                             {
                                                                                 "Value":  "order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU",
                                                                                 "Name":  "QueueName"
                                                                             }
                                                                         ],
                                                          "EvaluationPeriods":  1,
                                                          "Namespace":  "AWS/SQS",
                                                          "AlarmDescription":  "Workshop 5.6; source stack: order-orchestration-evidence",
                                                          "TreatMissingData":  "notBreaching",
                                                          "MetricName":  "ApproximateNumberOfMessagesVisible",
                                                          "Statistic":  "Maximum",
                                                          "Threshold":  0,
                                                          "DatapointsToAlarm":  1,
                                                          "Period":  60
                                                      },
                                       "Type":  "AWS::CloudWatch::Alarm"
                                   },
                      "LambdaErrorAlarm":  {
                                               "Properties":  {
                                                                  "AlarmName":  "order-monitoring-evidence-lambda-errors",
                                                                  "ComparisonOperator":  "GreaterThanThreshold",
                                                                  "OKActions":  [
                                                                                    "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                                ],
                                                                  "AlarmActions":  [
                                                                                       "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                                   ],
                                                                  "Dimensions":  [
                                                                                     {
                                                                                         "Value":  "order-orchestration-evidence-ProcessOrderFn-A3d416bkFPjD",
                                                                                         "Name":  "FunctionName"
                                                                                     }
                                                                                 ],
                                                                  "EvaluationPeriods":  1,
                                                                  "Namespace":  "AWS/Lambda",
                                                                  "AlarmDescription":  "Workshop 5.6; source stack: order-orchestration-evidence",
                                                                  "TreatMissingData":  "notBreaching",
                                                                  "MetricName":  "Errors",
                                                                  "Statistic":  "Sum",
                                                                  "Threshold":  0,
                                                                  "DatapointsToAlarm":  1,
                                                                  "Period":  60
                                                              },
                                               "Type":  "AWS::CloudWatch::Alarm"
                                           }
                  }
}
```

Redeploy the same stack using the file in Downloads:

```powershell
$template = Join-Path $env:USERPROFILE 'Downloads\workshop-monitoring-5-6.template.json'
aws cloudformation deploy --template-file $template --stack-name order-monitoring-evidence --region ap-southeast-1 --no-fail-on-empty-changeset --no-cli-pager
```

## 2. Four original deployment screenshots

**H1.** Alarm/dashboard resources are CREATE_COMPLETE and order-system is UPDATE_COMPLETE, establishing successful deployment.

![H1 - Original monitoring deployment](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h1.png)

**H2.** The original alarms show OK and Actions enabled. This alone does not establish received email notifications.

![H2 - Original three alarm states](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h2.png)

**H3.** order-workshop displays queues, message age, Step Functions and Lambda metrics.

![H3 - Original dashboard](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h3.png)

**H4.** order-sfn-failed is ALARM with an update timestamp of October 31, 2026, 12:40:46 UTC. This timestamp is not used to measure latency for the new run.

![H4 - Original Step Functions alarm](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h4.png)

## 3. Experiment procedure

Prerequisites are AWS CLI v2, a stable source stack, a verified SES sender and a confirmed SNS email subscription. Use Downloads/workshop-monitoring-5-6.json generated during setup. The script below records initial alarm/DLQ state and sends three messages with fresh IDs on each run.

| Case | Input | Expected observations to check |
|---|---|---|
| Success | Valid SES simulator recipient | SUCCEEDED execution and DONE order |
| Email failure | not-an-email | Step Functions failure/retries, FAILED execution and sfn-failed alarm |
| Malformed JSON | not-json-mon56-... string | Lambda JSON parsing failure, SQS retries and DLQ arrival |

With the section 5.5 code, Lambda returns after starting the workflow. A later SES failure need not increase Lambda Errors. Malformed JSON independently tests Lambda and the DLQ. An SQS MessageId proves acceptance, not processing completion.

```powershell
& {
  param([string]$ConfigPath = (Join-Path $env:USERPROFILE 'Downloads\workshop-monitoring-5-6.json'))
  $ErrorActionPreference = 'Stop'
  function Read-Aws([string[]]$Arguments) {
    $raw = & aws @Arguments --region $c.Region --output json --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw ('AWS CLI failed: '+($Arguments -join ' ')) }
    return (($raw -join [Environment]::NewLine) | ConvertFrom-Json)
  }
  try {
    $c = Get-Content -LiteralPath $ConfigPath -Raw | ConvertFrom-Json
    $id = 'mon56-'+(Get-Date -Format 'yyyyMMdd-HHmmss')+'-'+[guid]::NewGuid().ToString('N').Substring(0,6)
    $dir = Join-Path (Split-Path -Parent $ConfigPath) $id
    $null = New-Item -ItemType Directory -Path $dir
    $utf8 = New-Object System.Text.UTF8Encoding($false)
    $before = Read-Aws -Arguments (@('cloudwatch','describe-alarms','--alarm-names') + @($c.AlarmNames))
    [IO.File]::WriteAllText((Join-Path $dir 'alarms-before.json'),($before | ConvertTo-Json -Depth 30),$utf8)
    $queueBefore = Read-Aws -Arguments @('sqs','get-queue-attributes','--queue-url',$c.DlqUrl,'--attribute-names','ApproximateNumberOfMessages','ApproximateNumberOfMessagesNotVisible')
    [IO.File]::WriteAllText((Join-Path $dir 'dlq-before.json'),($queueBefore | ConvertTo-Json -Depth 10),$utf8)
    Write-Host '=== ALARMS BEFORE TEST ===' -ForegroundColor Cyan
    $before.MetricAlarms | Select-Object AlarmName,StateValue | Format-Table -AutoSize
    $good = @{orderId=($id+'-ok');customerEmail='success@simulator.amazonses.com';acceptedAt=[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()}
    $bad = @{orderId=($id+'-ses-fail');customerEmail='not-an-email';acceptedAt=[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()}
    $cases = @(
      @{Name='success';Body=($good | ConvertTo-Json -Compress)},
      @{Name='ses-failure';Body=($bad | ConvertTo-Json -Compress)},
      @{Name='malformed-json';Body=('not-json-'+$id)}
    )
    $results = @()
    foreach ($case in $cases) {
      $file = Join-Path $dir ($case.Name+'.txt')
      [IO.File]::WriteAllText($file,$case.Body,$utf8)
      $sent = Read-Aws -Arguments @('sqs','send-message','--queue-url',$c.QueueUrl,'--message-body',('file://'+$file))
      $results += [pscustomobject]@{Case=$case.Name;MessageId=$sent.MessageId;SentAtUtc=[DateTimeOffset]::UtcNow.ToString('o')}
      [IO.File]::WriteAllText((Join-Path $dir 'sent.json'),($results | ConvertTo-Json -Depth 10),$utf8)
    }
    $run = @{RunId=$id;GoodOrderId=$good.orderId;FailedOrderId=$bad.orderId;MalformedBody=('not-json-'+$id);EvidenceDirectory=$dir}
    [IO.File]::WriteAllText((Join-Path $dir 'run.json'),($run | ConvertTo-Json -Depth 10),$utf8)
    Write-Host '=== TEST MESSAGES ACCEPTED BY SQS ===' -ForegroundColor Green
    $results | Format-Table -AutoSize
    $run | Format-List
    Write-Host 'Da gui 3 tin; chua ket luan xu ly thanh cong. Theo doi Step Functions, Lambda, DLQ va CloudWatch trong 5-10 phut.'
    Write-Host ('Lambda logs: aws logs tail "'+$c.LogGroup+'" --since 15m --region '+$c.Region+' --format short --no-cli-pager')
    Write-Host 'DLQ co san tin cu thi ALARM/email cu khong chung minh do tin vua gui.'
  } catch {
    Write-Host ('LOI: '+$_.Exception.Message) -ForegroundColor Red
    Write-Host $_.InvocationInfo.PositionMessage -ForegroundColor Yellow
  }
}
```

Observe for an initial 5–10 minutes and allow additional time for retries and metric publication. Find executions by orderId; correlate Lambda logs, the DLQ queue/body, alarm Details/History and matching email subjects. Do not use set-alarm-state as evidence of an actual application failure.

## 4. Observations from H5–H11

The test message contains run ID **mon56-20261114-040056-316515**. Dashboard/alarm times are UTC; the inbox and SQS timestamps shown use UTC+07:00. For example, November 13, 2026, 21:03:44 UTC equals November 14, 2026, 04:03:44 UTC+07:00.

**H5. Supplementary dashboard.** All four widgets and legends are visible; the DLQ already shows a value of 1 before the test. ExecutionsFailed is still 0 at this capture, while H6 records a failure in a separate screenshot. H5 is not the final outcome of all executions.

![H5 - Supplementary dashboard](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h5.png)

**H6. Step Functions.** order-monitoring-evidence-sfn-failed is ALARM and ExecutionsFailed reaches 1.

![H6 - Step Functions alarm](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h6.png)

**H7. Lambda.** order-monitoring-evidence-lambda-errors is ALARM and Errors reaches 1. The metric screenshot alone does not identify the cause or MessageId of the failed invocation.

![H7 - Lambda Errors alarm](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h7.png)

**H8. DLQ.** order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU contains not-json-mon56-20261114-040056-316515. An older message is also visible. Receive count is not treated as an exact Lambda attempt count because console receives can also affect it.

![H8 - Test message in the DLQ](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h8.png)

**H9. Step Functions ALARM email.** The email records datapoint 1 at November 13, 2026, 21:02 UTC and transition to ALARM at 21:03:44 UTC. This establishes receipt of a CloudWatch notification. The screenshots lack the corresponding execution failure timestamp needed to calculate detection latency.

![H9 - Step Functions ALARM email](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h9.png)

**Lambda ALARM email.** The CloudWatch email reports an Errors datapoint of 1 at November 13, 2026, 21:01:00 UTC, exceeding the threshold of 0. Alarm order-monitoring-evidence-lambda-errors transitions from OK to ALARM at 21:02:21 UTC, corresponding to November 14, 2026, 04:02:21 UTC+07:00. The inbox screenshot establishes receipt of the ALARM notification, consistent with the CloudWatch alarm state. The datapoint timestamp is not the exact Lambda failure time and is not used to calculate detection or email-delivery latency.

![Lambda ALARM email](/images/5-Workshop/5.6-Monitoring/lambda-alarm-20261114-044826-a961f7/h10.png)

**H11. DLQ ALARM email.** It records datapoint 1 at November 13, 2026, 20:59 UTC and ALARM at 21:01:06 UTC. That datapoint predates the newly sent message in H8. Interpret it as notification of existing backlog, not a transition caused specifically by the new message.

![H11 - DLQ backlog ALARM email](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h11.png)

## 5. Assessment and limits

| Area | Supported conclusion |
|---|---|
| Dashboard | Four metric groups are visible (H5) |
| Step Functions / Lambda alarms | Both are observed in ALARM (H6, H7) |
| New test message in DLQ | Matching body and queue are visible (H8) |
| Step Functions / DLQ ALARM emails | Received (H9, H11); DLQ notification concerns backlog |
| Lambda ALARM email | ALARM notification received, matching the alarm name and Errors = 1 datapoint |
| Recovery and alert latency | Insufficient evidence to establish recovery or calculate duration |

Polling makes DLQ messages temporarily invisible and can reduce visible counts without resolution/deletion. OK with notBreaching does not independently establish application health. Retain run.json, sent.json, alarms-before.json and dlq-before.json for correlation; do not infer exact order totals or individual outcomes solely from metrics.

## 6. Cost and cleanup

The additional alarms and dashboard may incur charges. After retaining evidence, delete only order-monitoring-evidence when no longer needed. The application stack and shared SNS topic remain. The section 5.7 order-orchestration-evidence-dlq-proof alarm is outside this stack.

## Conclusion

The evidence records a dashboard, Step Functions/Lambda alarms, the test message in the DLQ, and received ALARM emails for all three alarms: Step Functions, Lambda and DLQ. The DLQ notification reflects backlog that predates the test. The screenshots do not establish post-failure recovery or alert latency measured from the actual failure time.