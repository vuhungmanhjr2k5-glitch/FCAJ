---
title: "Write the Lambda code"
weight: 2
chapter: false
pre: " <b> 5.4.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Write the application handler

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

```powershell
$content = @'
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async (event) => {
  for (const record of event.Records) {
    const order = JSON.parse(record.body); // invalid JSON throws an error -> retry -> DLQ

    try {
      await db.send(new PutCommand({
        TableName: process.env.TABLE_NAME,
        Item: {
          orderId: order.orderId,
          status: "PROCESSING",
          createdAt: Date.now(),
        },
        // Only write if the orderId does not exist yet
        ConditionExpression: "attribute_not_exists(orderId)",
      }));
    } catch (e) {
      if (e.name === "ConditionalCheckFailedException") {
        console.log("Duplicate order, skipping:", order.orderId);
        continue;
      }
      throw e;
    }

    console.log("Processed:", order.orderId);
  }
};
'@
$path = Join-Path $PWD.Path 'src/index.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

The key point is the `ConditionExpression`: DynamoDB only allows the write if
the `orderId` does not already exist, so the second write is rejected and the
Lambda skips that order.

> In section 5.5 we will add an order status (`PROCESSING` → `DONE`) to handle
> the case where the order was recorded in the ledger but the email-sending
> step failed.

### Package the worker dependencies

```powershell
Push-Location -LiteralPath .\src -ErrorAction Stop
try {
    if (-not (Test-Path -LiteralPath .\package.json)) {
        npm.cmd init -y
        if ($LASTEXITCODE -ne 0) { throw 'npm init failed.' }
    }
    npm.cmd install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
    if ($LASTEXITCODE -ne 0) { throw 'Dependency installation failed.' }
} finally {
    Pop-Location
}
```

## Put the condition on the write itself

Imagine two invocations receiving `A001`. A separate read followed by a write would let both check before either item exists. DynamoDB evaluates `attribute_not_exists(orderId)` as part of Put. One write wins and the other receives `ConditionalCheckFailedException`. The check applies to the target item, so unrelated order IDs remain independent.

Read the catch branches separately. A condition failure logs the ID and lets the batch complete. The SQS integration can acknowledge processing. Another exception propagates out of the worker so the message can be delivered again. Treating a denied DynamoDB write as a duplicate could acknowledge a new order that had never been recorded.

## Establish the boundary of this worker

After the first successful Put, the item contains `createdAt` and `PROCESSING`. If the process stops immediately afterward, the next delivery finds an existing key. This version has no workflow to resume an email operation, so the result proves that an order marker was written. Section 5.5 introduces `RECEIVED` and a named execution to handle startup after recording.

Check the JavaScript syntax locally after writing the file. This check does not invoke the handler or consume a message.

```powershell
node --check .\src\index.mjs
```
