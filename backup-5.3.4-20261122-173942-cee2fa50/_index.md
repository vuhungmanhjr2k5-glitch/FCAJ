---
title: "Test a message falling into the DLQ"
weight: 4
chapter: false
pre: " <b> 5.3.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Store the two queue URLs in variables:

```powershell
$stackData = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read order-system.' }
$outputs = @{}
foreach ($item in ($stackData | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$q = $outputs['QueueUrl']
if (-not $q) { throw 'Missing output: QueueUrl' }
$dlq = $outputs['DlqUrl']
if (-not $dlq) { throw 'Missing output: DlqUrl' }
```

Send a test message:

```powershell
$testBody = "test-dlq-$([guid]::NewGuid().ToString('N').Substring(0,10))"
$testBody
aws sqs send-message --queue-url $q --message-body $testBody --region ap-southeast-1 --output json --no-cli-pager
```

Receive the message several times **without deleting it** (the `--visibility-timeout 0` parameter makes the message visible again immediately):

```powershell
1..4 | ForEach-Object {
  aws sqs receive-message --queue-url $q --visibility-timeout 0 --wait-time-seconds 2 --message-system-attribute-names ApproximateReceiveCount --region ap-southeast-1 --output json --no-cli-pager
  Start-Sleep -Seconds 1
}
```

SQS can move the message once actual receives cross the threshold. Wait about a minute, then check the DLQ:

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages --region ap-southeast-1 --output json --no-cli-pager
```

Or go to the Console → the DLQ queue → **Send and receive messages** → **Poll for messages**.

![Message in the DLQ](/images/5-Workshop/5.3-SQS-DLQ/dlq-message.png)

> SQS metrics are approximate and may be delayed by several tens of seconds.

## Give the experiment a recognizable message

Use the lab queues before attaching the worker in section 5.4. An active consumer could process and delete the test message before your manual poll. Give the body a run-specific marker and keep the `MessageId` returned by SQS. An older DLQ item can make its count positive even before this run.

The receive loop sets the visibility timeout to zero for these individual receives. That makes redelivery possible quickly while leaving the source queue's configured 60 seconds intact. Leave the message undeleted. Each response containing the matching body counts as an observed delivery. An empty poll is also possible with a Standard queue.

## Interpret the end of the polling loop

If the new body appears in the DLQ, record the run marker, message ID and destination name. If the source still contains it, check how many matching deliveries occurred and inspect the redrive policy before polling again. If both visible counts temporarily read zero, allow for metric lag. Polling the DLQ to inspect a body is itself a receive operation and can hide the item for a visibility interval.

This experiment establishes redelivery behavior for one message. In the next chapter a failing Lambda invocation supplies the reason for retry. Here the manual receive without deletion supplies that condition.
