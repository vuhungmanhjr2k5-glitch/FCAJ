---
title: "Replace the entire src/index.mjs"
weight: 2
chapter: false
pre: " <b> 5.5.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Write the application handler

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

```powershell
$content = @'
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  PutCommand,
  GetCommand,
} from "@aws-sdk/lib-dynamodb";
import { SFNClient, StartExecutionCommand } from "@aws-sdk/client-sfn";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const sfn = new SFNClient({});

export const handler = async (event) => {
  for (const record of event.Records) {
    const order = JSON.parse(record.body); // invalid JSON -> error -> retry -> DLQ

    // Step 1: write to the ledger; each orderId may only be written once
    try {
      await db.send(new PutCommand({
        TableName: process.env.TABLE_NAME,
        Item: { orderId: order.orderId, status: "RECEIVED", createdAt: Date.now() },
        ConditionExpression: "attribute_not_exists(orderId)",
      }));
    } catch (e) {
      if (e.name !== "ConditionalCheckFailedException") throw e;

      const { Item } = await db.send(new GetCommand({
        TableName: process.env.TABLE_NAME,
        Key: { orderId: order.orderId },
        ConsistentRead: true,
      }));
      if (Item?.status !== "RECEIVED") {
        console.log("Duplicate order, skipping:", order.orderId);
        continue;
      }
      // Still in RECEIVED state: the previous attempt failed to start the workflow -> retry
    }

    // Step 2: start the workflow. The execution name = orderId, so it cannot run twice.
    try {
      await sfn.send(new StartExecutionCommand({
        stateMachineArn: process.env.STATE_MACHINE_ARN,
        name: order.orderId,
        input: JSON.stringify(order),
      }));
      console.log("Workflow started:", order.orderId);
    } catch (e) {
      if (e.name === "ExecutionAlreadyExists") {
        console.log("Workflow already exists, skipping:", order.orderId);
        continue;
      }
      throw e;
    }
  }
};
'@
$path = Join-Path $PWD.Path 'src/index.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

> Because `orderId` is used as the execution name, it should only contain letters,
> digits, `-`, and `_` (up to 80 characters).

### Package the worker dependencies

```powershell
Push-Location -LiteralPath .\src -ErrorAction Stop
try {
    if (-not (Test-Path -LiteralPath .\package.json)) {
        npm.cmd init -y
        if ($LASTEXITCODE -ne 0) { throw 'npm init failed.' }
    }
    npm.cmd install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb @aws-sdk/client-sfn
    if ($LASTEXITCODE -ne 0) { throw 'Dependency installation failed.' }
} finally {
    Pop-Location
}
```

## Revisit a recorded order that is still RECEIVED

The worker performs two separate operations: recording an order and asking Step Functions to start an execution. If the process stops between them, the item can still be `RECEIVED` on redelivery. After a conditional conflict, the handler reads that item. It attempts startup for `RECEIVED` and can acknowledge a terminal `DONE` or `FAILED` item. The example uses a strongly consistent key read after conflict to observe the completed write.

The execution name comes from `orderId`. For a running Standard workflow, StartExecution with the same name and input can return the existing execution. Changing the input under the same name can produce `ExecutionAlreadyExists`. Preserve the business payload when replaying a purchase. Amending a confirmed purchase requires a separately defined operation.

## Trace three arrivals through the handler

| Item at arrival | Handler path | Observation |
|---|---|---|
| No item | Conditional Put succeeds, then StartExecution | An execution named after the ID |
| Existing RECEIVED | Read state and attempt startup under the same name | History of the named execution |
| Existing terminal state | Acknowledge the redelivery | Matching log and unchanged terminal state |

The name constrains startup under the naming and retention rules of Standard workflows. The downstream states record completion and any application counter. Counting “started” log lines alone cannot tell how many emails arrived.
