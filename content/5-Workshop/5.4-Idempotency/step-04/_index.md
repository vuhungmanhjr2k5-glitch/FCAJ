---
title: "Test"
weight: 4
chapter: false
pre: " <b> 5.4.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Set up the order ID and lookup targets

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
$table = $outputs['TableName']
if (-not $table) { throw 'Missing output: TableName' }
```

### 1. Submit an order and read by key

```powershell
$orderId = 'A-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$file = Join-Path $PWD.Path ("{0}.json" -f $orderId)
$body = @{ orderId = $orderId } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
```

Allow the worker time to process. If the first read has no `Item`, repeat just the read block.

```powershell
$keyFile = Join-Path $PWD.Path 'workshop-item-key.json'
$keyJson = @{ orderId = @{ S = $orderId } } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($keyFile, $keyJson, [Text.UTF8Encoding]::new($false))
aws dynamodb get-item --table-name $table --key "file://$keyFile" --consistent-read --region ap-southeast-1 --output json --no-cli-pager
```

### 2. Resubmit the same saved payload

```powershell
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
sam logs --stack-name order-system --name ProcessOrderFn --region ap-southeast-1 --tail
```

Wait for the duplicate log naming `$orderId`. Press **Ctrl+C** to end log tailing.

### 3. Trace an unreadable body

```powershell
$badBody = "not-json-$orderId"
aws sqs send-message --queue-url $q --message-body $badBody --region ap-southeast-1 --output json --no-cli-pager
```

Allow several minutes for redelivery with the lab settings. Inspect the DLQ identified by `$dlq` for the exact body. In the recorded `order-idempotency-evidence` run the malformed body was `khong-phai-json` and the valid order was `A001`.

## Sequence the duplicate test so it is interpretable

The first send creates the item. Wait until `PROCESSING` is visible and note `createdAt` before sending the second request. Reuse the same JSON file. Changing the ID tests two valid orders. Conversely, reusing `A001` from an older session can send even the first request down the duplicate branch. Choose a new ID for an independent run.

One DynamoDB row after two sends can also result from an unconditional overwrite to the same key. Pair a log line identifying the duplicate with the unchanged creation time. This version produces that combination when a conditional write succeeds and the next condition failure is handled.

## Separate a parse failure from a table write failure

The plain-text body in case 3 fails at JSON parsing before the worker can extract an `orderId`. Correlate it with the DLQ using a body marker and send time. A denied DynamoDB write can also lead to retries, but its log carries a different exception or action. Read the first error before editing the table settings. Record the observed body or matching log next to the run identifier.

### Read a typed item in AWS CLI output

In GetItem output, `orderId.S` contains a string ID, `status.S` contains `PROCESSING` and `createdAt.N` contains a number represented as text by the DynamoDB API. Two reads with the same key and creation value indicate the initial item survived. Pair this observation with the duplicate log to identify the worker decision on the second submission.

### Illustrations from the recorded run

- [Data in DynamoDB](/images/5-Workshop/5.4-Idempotency/dynamodbb-item.png)
- [Lambda skips the resubmitted order A001](/images/5-Workshop/5.4-Idempotency/duplicate-logg.png)
- [DynamoDB retains one A001 record after resubmission](/images/5-Workshop/5.4-Idempotency/duplicate-item.png)
- [Invalid message found in the DLQ](/images/5-Workshop/5.4-Idempotency/dlqq-message.png)
