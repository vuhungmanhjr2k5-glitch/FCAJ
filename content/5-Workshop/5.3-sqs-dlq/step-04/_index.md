---
title: "Test a message falling into the DLQ"
weight: 4
chapter: false
pre: " <b> 5.3.4 </b> "
---

[← Section contents](../)

## Purpose and prerequisites

This experiment simulates a consumer receiving a message without deleting it. When actual receives exceed the configured redrive threshold, SQS can move the message to the DLQ for investigation.

Use the lab queues before attaching Lambda in section 5.4. An active consumer may receive or delete the message before the manual operation. That introduces competing consumers and changes the conditions of this test.

In SQS in Singapore, inspect the source queue's **Dead-letter queue** tab. Confirm that it references the intended OrdersDLQ and that **Maximum receives = 3**. A configuration screenshot proves the redrive policy, not a completed message transfer.

## 1. Retrieve both queue URLs

Run the following blocks in the same PowerShell session. Read outputs from the intended stack to avoid inspecting another queue.

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
[pscustomobject]@{ QueueUrl = $q; DlqUrl = $dlq } | Format-List
```

Check the DLQ count before submitting a message to identify any existing backlog:

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read DLQ attributes.' }
```

## 2. Send an identifiable test message

The body includes a random suffix to distinguish independent runs. Save both Body and MessageId from the output for later correlation.

```powershell
$testBody = "test-dlq-$([guid]::NewGuid().ToString('N').Substring(0,10))"
$sendResult = aws sqs send-message --queue-url $q --message-body $testBody --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot send test message.' }
$sent = $sendResult | Out-String | ConvertFrom-Json
[pscustomobject]@{ Body = $testBody; MessageId = $sent.MessageId } | Format-List
```

## 3. Receive the message without deleting it

The per-receive `--visibility-timeout 0` makes the message eligible for receipt again quickly. It does not change the source queue's default 60-second visibility setting. Do not call DeleteMessage in the loop.

```powershell
1..4 | ForEach-Object {
    Write-Host "Receive attempt $_"
    aws sqs receive-message --queue-url $q --visibility-timeout 0 --wait-time-seconds 2 --message-system-attribute-names ApproximateReceiveCount --region ap-southeast-1 --output json --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw 'Cannot receive messages.' }
    Start-Sleep -Seconds 1
}
```

Four command calls do not guarantee four deliveries of the same message. A Standard queue can return an empty poll. Correlate successful receives with the test Body or MessageId and inspect ApproximateReceiveCount when present.

## 4. Inspect the DLQ and correlate results

Wait approximately one minute and run the following command. This is an initial observation interval, not a guarantee that transfer will finish within one minute.

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read DLQ attributes.' }
```

To inspect the content, open **SQS → the intended DLQ → Send and receive messages → Poll for messages**. Open the message and compare its Body or MessageId with the saved values. Include the DLQ name and message content together in any screenshot.

> SQS counts are approximate and can be delayed. A count of 1 may reflect existing backlog; an empty receive does not establish that the message moved to the DLQ. Console polling also receives messages and may temporarily make them invisible.

## Evidence from an earlier test

![Repeated message receives and a destination queue count query](../../../images/5-Workshop/5.3-SQS-DLQ/dlq-receive-result.png)

*PowerShell shows the same MessageId with Body `test-dlq` in receive attempts 2 and 3. After a waiting period, the queue referenced by `$dlq` reports `ApproximateNumberOfMessages = 1`.*

The screenshot comes from the earlier version using the fixed Body `test-dlq`. The current instructions use `test-dlq-<random suffix>` to distinguish runs. The image illustrates interpretation of that earlier test; it is not output from the newly generated identifier.

The image supports repeated delivery and inspection of the destination count. To establish that the intended message reached the DLQ, also correlate the destination queue name with its Body or MessageId. Receive count 4 or a positive DLQ count alone is insufficient.

## Recording the outcome

- If the matching message is found in the DLQ, record its Body, MessageId, DLQ name and observation time.
- If it remains in the source queue, inspect actual receives and the redrive policy before polling again.
- If both counts temporarily show 0, allow updates and examine receive results before concluding that a message is lost.
- If destination or message evidence is missing, record the outcome as not fully confirmed.

This experiment concerns receiving and leaving a message undeleted. In the later Lambda exercise, processing failures can cause redelivery; evaluate that scenario separately.
