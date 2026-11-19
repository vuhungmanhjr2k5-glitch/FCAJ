---
title: "Testing"
weight: 4
chapter: false
pre: " <b> 5.5.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Prepare a test run

```powershell
$stackData = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read order-system.' }
$outputs = @{}
foreach ($item in ($stackData | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$q = $outputs['QueueUrl']
if (-not $q) { throw 'Missing output: QueueUrl' }
$table = $outputs['TableName']
if (-not $table) { throw 'Missing output: TableName' }
$tag = [guid]::NewGuid().ToString('N').Substring(0,10)
$goodId = "B001-$tag"
$failId = "B002-$tag"
$email = Read-Host 'Verified test inbox for the order confirmation'
```

### 1. Trace the successful order

```powershell
$file = Join-Path $PWD.Path ("{0}.json" -f $goodId)
$body = @{ orderId = $goodId; customerEmail = $email } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
$goodFile = $file
$goodId
```

Find `$goodId` in Step Functions and then read its item. After the execution succeeds, check for `DONE` and a matching email subject.

```powershell
$keyFile = Join-Path $PWD.Path 'workshop-item-key.json'
$keyJson = @{ orderId = @{ S = $goodId } } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($keyFile, $keyJson, [Text.UTF8Encoding]::new($false))
aws dynamodb get-item --table-name $table --key "file://$keyFile" --consistent-read --region ap-southeast-1 --output json --no-cli-pager
```

### 2. Resubmit the same body

```powershell
aws sqs send-message --queue-url $q --message-body "file://$goodFile" --region ap-southeast-1 --output json --no-cli-pager
sam logs --stack-name order-system --name ProcessOrderFn --region ap-southeast-1 --tail
```

Wait for the duplicate log and press **Ctrl+C**.

### 3. Follow a rejected SES task

In the SES sandbox an unverified normal recipient is rejected. This block uses a malformed address to give a failure case even with production access.

```powershell
$email = 'not-an-email'
$file = Join-Path $PWD.Path ("{0}.json" -f $failId)
$body = @{ orderId = $failId; customerEmail = $email } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
$failId
```

Find execution `$failId`. After retries inspect `MarkFailed`, `NotifyAdminFailure`, the `FAILED` item and the administrative email for that ID. The original figures show `B001` and `B002`.

## Keep the three paths tied to their own IDs

Use a fresh ID for the success case and another for the SES rejection. `B001` and `B002` in the illustrations belong to the recorded run. The PowerShell blocks use a suffix so a repeat session can start new executions. Save the successful payload and use that same payload for the duplicate submission.

For a valid order distinguish SQS returning a message ID, the execution reaching Succeeded, and the table showing `DONE`. The 5.5 template updates `status` at completion. An extended deployment illustrated here can also show `doneAt` and `emailCount`. Those fields enter the lab template in 5.7.1. A verified real inbox can demonstrate delivery. The success simulator provides no inbox to inspect.

## Read the rejected order's timeline

In **Execution event history**, open the first failed SES task and record the error and time. Find the next scheduling of that task and follow the route to `MarkFailed`. For a failed execution, also check the preceding SNS publish. The `FAILED` item establishes that the table update succeeded. A successful publish establishes acceptance by the topic. A received email carrying the matching ID establishes arrival at the operator's inbox.

Use workflow history for this case. An SES failure after the worker successfully started the workflow occurs downstream of SQS delivery.

### Keep a record while waiting for the execution

| Field | Successful order | SES failure |
|---|---|---|
| ID to inspect | Value of `$goodId` | Value of `$failId` |
| State events | Successful SES task and `MarkDone` update | First SES failure and entry to `MarkFailed` |
| Terminal history | `ExecutionSucceeded` | `ExecutionFailed` after the administrative path |
| External observation | Item by key and confirmation in the test inbox | Item by key and SNS notification for the order |

If the first check finds only `ExecutionStarted`, refresh the same execution. For a terminal event with an unexpected table status, check the fresh order key, table and Region. For a successful email task and an empty inbox, inspect SES acceptance and the recipient type before changing the deployment. Record time zones for history and inbox timestamps.

### Illustrations from the recorded run

- [Execution B001 has status Succeeded](/images/5-Workshop/5.5-Orchestration/b001-execution-succeeded.png)
- [Graph view showing successful SendCustomerEmail and MarkDone steps](/images/5-Workshop/5.5-Orchestration/b001-workflow-graph.png)
- [DynamoDB records order B001 with status DONE](/images/5-Workshop/5.5-Orchestration/b001-dynamodb-done.png)
- [Successful execution](/images/5-Workshop/5.5-Orchestration/execution-success.png)
- [Failed execution](/images/5-Workshop/5.5-Orchestration/execution-failed.png)
- [Alert email](/images/5-Workshop/5.5-Orchestration/alert-email.png)
