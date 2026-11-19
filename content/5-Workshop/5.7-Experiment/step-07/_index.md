---
title: "Experiment 3: failures and alerts"
weight: 7
chapter: false
pre: " <b> 5.7.7 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

### SES failure and retries

Order **failure-proof-20261113-222244-4343a2** used **not-an-email**. SES returned **SesV2.BadRequestException**, HTTP **400**, cause **Missing final '@domain'**. Event history shows task rescheduling and execution after failures. SDK Attempt Count: 1 is not the total Step Functions retry count.

![SES error cause and retry history](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-ses-error-history.png)

### Failure branch

Catch routed to **MarkFailed** and **NotifyAdminFailure**, both successful. The workflow terminated at **OrderFailed**; MarkDone was skipped.

[Workflow failure-handling branch](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-workflow-graph.png)

The SNS email contains the matching order ID and was sent by NotifyAdminFailure.

[Alert email from the workflow](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-admin-email.png)

### DLQ and CloudWatch Alarm

A separate test used invalid JSON: **not-json-dlq-proof-20261113-223327-eed369**. Inspect the message in **order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU**; the screenshot should show both the queue name and message body.

[Malformed message in the identified DLQ](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-malformed-message.png)

Alarm **order-orchestration-evidence-dlq-proof** monitored the DLQ ApproximateNumberOfMessagesVisible metric and entered **ALARM** at **November 13, 2026, 15:39:57 UTC**.

[Triggered DLQ alarm](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-cloudwatch-alarm.png)

The CloudWatch email delivered through SNS records the same alarm and transition time, with datapoint **1** exceeding threshold **0**. This is distinct from the workflow notification above. These screenshots do not measure alert latency.

[CloudWatch alarm email via SNS](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-alarm-email.png)

## Follow two distinct failure paths

`not-an-email` is still a string inside valid JSON. The worker can record the order and start an execution before SES validates the address and rejects it. A `not-json-...` body fails at the worker parser and never reaches the email state. Keep separate markers so you do not search for a workflow execution that could never start.

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue | [0]" --region ap-southeast-1 --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $q -eq 'None') { throw 'QueueUrl unavailable.' }
$tag = [guid]::NewGuid().ToString('N').Substring(0,10)
$failedId = "ses-fail-$tag"
$badBody = "not-json-$tag"
$file = Join-Path $PWD.Path "$failedId.json"
$body = @{ orderId = $failedId; customerEmail = 'not-an-email' } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --query MessageId --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'First send failed.' }
aws sqs send-message --queue-url $q --message-body $badBody --region ap-southeast-1 --query MessageId --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Second send failed.' }
[pscustomobject]@{ OrderId = $failedId; DlqMarker = $badBody }
```

For the first case, inspect the first error, task reschedules, table update and `NotifyAdminFailure` publish. The 5.7 template permits six retries, so allow for backoff before checking the terminal state. A recorded environment may have different retry settings. Count events for the run you actually inspect. For the second case, find `$badBody` in the application's DLQ and check the corresponding alarm from 5.6.

## Capture comparable timestamps

Separating detection time and mail delivery needs the initial error from history or a log, the alarm transition and the inbox arrival. Convert all three to UTC before subtracting. The SNS notification timestamp describes a service event and may differ from mailbox receipt. If the DLQ was already in alarm, the new item can arrive without a new state-transition email. Record that as a valid observation of the configured alarm behavior.
