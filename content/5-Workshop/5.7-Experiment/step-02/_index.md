---
title: "Write the code"
weight: 2
chapter: false
pre: " <b> 5.7.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

## Synchronous HTTP function

```powershell
$content = @'
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, UpdateCommand } from "@aws-sdk/lib-dynamodb";
import { SESv2Client, SendEmailCommand } from "@aws-sdk/client-sesv2";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const ses = new SESv2Client({});
const ID = /^[A-Za-z0-9_-]{1,64}$/;

export const handler = async (event) => {
  if (event.headers?.["x-api-key"] !== process.env.API_KEY)
    return { statusCode: 403, body: "forbidden" };

  let order;
  try { order = JSON.parse(event.body); }
  catch { return { statusCode: 400, body: "invalid json" }; }
  if (typeof order?.orderId !== "string" || !ID.test(order.orderId) || !order.customerEmail)
    return { statusCode: 400, body: "invalid order" };

  const key = { orderId: order.orderId };
  try {
    await db.send(new UpdateCommand({
      TableName: process.env.TABLE_NAME,
      Key: key,
      UpdateExpression: "SET #s = :r, createdAt = if_not_exists(createdAt, :c)",
      ExpressionAttributeNames: { "#s": "status" },
      ExpressionAttributeValues: { ":r": "RECEIVED", ":c": Date.now() },
    }));

    await ses.send(new SendEmailCommand({
      FromEmailAddress: process.env.SENDER_EMAIL,
      Destination: { ToAddresses: [order.customerEmail] },
      Content: { Simple: {
        Subject: { Data: `Order confirmation ${order.orderId}` },
        Body: { Text: { Data: `Thank you. Your order ${order.orderId} has been received.` } },
      } },
    }));

    await db.send(new UpdateCommand({
      TableName: process.env.TABLE_NAME,
      Key: key,
      UpdateExpression: "SET #s = :d, doneAt = :t ADD emailCount :one",
      ExpressionAttributeNames: { "#s": "status" },
      ExpressionAttributeValues: { ":d": "DONE", ":t": new Date().toISOString(), ":one": 1 },
    }));
    return { statusCode: 200, body: JSON.stringify({ orderId: order.orderId, status: "DONE" }) };
  } catch (e) {
    console.error("Error processing order", order.orderId, e.name);
    return { statusCode: 500, body: "error" };
  }
};
'@
$path = Join-Path $PWD.Path 'src/sync-order.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Asynchronous intake function

```powershell
$content = @'
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";

const sqs = new SQSClient({});
const ID = /^[A-Za-z0-9_-]{1,64}$/;

export const handler = async (event) => {
  if (event.headers?.["x-api-key"] !== process.env.API_KEY)
    return { statusCode: 403, body: "forbidden" };

  let order;
  try { order = JSON.parse(event.body); }
  catch { return { statusCode: 400, body: "invalid json" }; }
  if (typeof order?.orderId !== "string" || !ID.test(order.orderId) || !order.customerEmail)
    return { statusCode: 400, body: "invalid order" };

  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.QUEUE_URL,
    MessageBody: JSON.stringify({
      orderId: order.orderId,
      customerEmail: order.customerEmail,
      acceptedAt: Date.now(),
    }),
  }));
  return { statusCode: 202, body: JSON.stringify({ orderId: order.orderId, status: "ACCEPTED" }) };
};
'@
$path = Join-Path $PWD.Path 'src/async-order.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Preserve the acceptance time in the worker

This is the complete 5.5.2 `src/index.mjs` worker with its updated `createdAt` value.

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
        Item: { orderId: order.orderId, status: "RECEIVED", createdAt: order.acceptedAt ?? Date.now() },
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

### Add the HTTP handlers’ SDK clients

```powershell
Push-Location -LiteralPath .\src -ErrorAction Stop
try {
    if (-not (Test-Path -LiteralPath .\package.json)) {
        npm.cmd init -y
        if ($LASTEXITCODE -ne 0) { throw 'npm init failed.' }
    }
    npm.cmd install @aws-sdk/client-sqs @aws-sdk/client-sesv2
    if ($LASTEXITCODE -ne 0) { throw 'Dependency installation failed.' }
} finally {
    Pop-Location
}
```

## Interpret each endpoint's response contract

Both handlers check the header key, parse JSON and validate the order identifier before touching the queue or table. Letters, digits, hyphens and underscores are suitable for the execution name. The sync success path returns 200 after completing its operations within the request. The async path returns 202 after SQS accepts the body. A 2xx response means the corresponding endpoint reached its own success boundary.

If SES rejects after the sync function wrote `RECEIVED`, the API may return 500 while an item exists. Report item presence and completed processing separately. For the async path, workflow failure can happen after the client has received 202. Query final states for the correct prefix to determine the outcome.

## Use a consistent timestamp convention

On `/async`, `acceptedAt` is recorded before the SQS send and accompanies the body to the worker. The worker uses it for `createdAt` when present. Earlier direct-to-queue test messages may omit that field, so the worker retains a current-time fallback. The sync handler records its start inside the request. Client-side API latency and the difference between stored application timestamps therefore have different starting boundaries.

Check the syntax of all three files after writing them. Dependency resolution and service permissions still need the subsequent build and small deployment test.

```powershell
node --check .\src\sync-order.mjs
node --check .\src\async-order.mjs
node --check .\src\index.mjs
```
