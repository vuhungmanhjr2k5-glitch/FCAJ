---
title: "Verify the result"
weight: 5
chapter: false
pre: " <b> 5.4.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

| Case | Expected result |
|---|---|
| Valid order | One row in DynamoDB |
| Duplicate send | The log reports a duplicate, no new row is added |
| Message that is not JSON | After a few retries, the message ends up in the DLQ |

## Make one record for each case

Fill in values from your own run. A `MessageId` traces an individual send, while `orderId` identifies the purchase. The duplicate case can yield two different message IDs for one order. For malformed input, record the body marker instead of an order ID. The case remains traceable even when the worker cannot create an item.

| Evidence | New order | Resubmission | Invalid body |
|---|---|---|---|
| Identity | Fresh order ID, first message ID | Same order ID, second message ID | Body marker and message ID |
| Table change | `createdAt` and `PROCESSING` appear | Compare with original creation time | Parsing stops before the write |
| Deciding observation | Processing log for this ID | Duplicate branch in the log | Parse error and matching body in the DLQ |

## Resolve incomplete observations

If the item exists but no duplicate log is visible, check the log window, stream and function. If the log is present but a table read appears stale, use a consistent read by key. An older body in the DLQ leaves the current malformed case pending. Record these results before advancing to orchestration, where the states acquire different meanings.

### Reopen a single order in a fresh PowerShell session

Enter the ID from the observation table. This request uses the primary key and strong consistency. For a response without `Item`, check the ID, table and Region before submitting another message.

```powershell
$orderId = Read-Host 'Order ID'
$table = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue | [0]" --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $table -eq 'None') { throw 'TableName unavailable.' }
$file = Join-Path $PWD.Path 'workshop-item-key.json'
$keyJson = @{ orderId = @{ S = $orderId } } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $keyJson, [Text.UTF8Encoding]::new($false))
aws dynamodb get-item --table-name $table --key "file://$file" --consistent-read --region ap-southeast-1 --output json --no-cli-pager
```
