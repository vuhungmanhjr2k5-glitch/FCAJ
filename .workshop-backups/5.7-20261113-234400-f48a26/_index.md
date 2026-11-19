---
title: "Fault and Load Experiments"
weight: 7
chapter: false
pre: " <b> 5.7 </b> "
---
## Objective

Compare two ways of processing orders under the same conditions:

| | Synchronous (baseline) | Asynchronous (proposed) |
|---|---|---|
| Flow | The API writes to DynamoDB and then sends via SES within the request | The API only puts the order into SQS; Lambda and Step Functions process it later |
| Duplicate protection | No | Yes (DynamoDB) |
| Retry, DLQ, alerts | No | Yes |

Metrics: API response time (average, p95), completed orders, failed or lost
orders, processing completion time, extra emails when duplicates are sent, and
the time until an alert is raised on failure.

>**SES sandbox limit:** at most 1 email/second. Emails sent to
>`success@simulator.amazonses.com` (the SES mailbox simulator) do not count
>toward the 200 emails/day quota but are still limited to 1 email/second. That
>is why the test loads are 20 and 50 orders. This is also a result worth
>recording: the bottleneck is the email-sending service, not the architecture.


#### 1. Update template.yaml

*a) Add a parameter* to the Parameters section:
```yaml
  ApiKey:
    Type: String
    NoEcho: true
    MinLength: 16
    Description: Secret key for the x-api-key header
```
*b) Modify the state machine.* In SendCustomerEmail, replace the Retry block so
it can tolerate SES rate limiting:
```yaml
            Retry:
              - ErrorEquals: [States.ALL]
                IntervalSeconds: 2
                MaxAttempts: 6
                BackoffRate: 2
                JitterStrategy: FULL
```
Replace the entire `MarkDone` state so it also records the completion time and
counts the emails sent:
```yaml
          MarkDone:
            Type: Task
            Resource: arn:aws:states:::dynamodb:updateItem
            Parameters:
              TableName: ${TableName}
              Key:
                orderId:
                  S.$: $.orderId
              UpdateExpression: "SET #s = :v, doneAt = :t ADD emailCount :one"
              ExpressionAttributeNames:
                "#s": status
              ExpressionAttributeValues:
                ":v":
                  S: DONE
                ":t":
                  S.$: $$.State.EnteredTime
                ":one":
                  N: "1"
            End: true
```
>With `MaxAttempts: 6` and a backoff rate of 2, an order with a permanent error
>(for example an unverified email) takes up to about 2 minutes to be marked
>`FAILED`, instead of the 20 to 30 seconds seen in section 5.5.


*c) Add two API functions* at the end of `Resources` (before `Outputs`):
```yaml
  SyncOrderFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: sync-order.handler
      Runtime: nodejs22.x
      CodeUri: src/
      MemorySize: 256
      Timeout: 15
      Environment:
        Variables:
          TABLE_NAME: !Ref OrdersTable
          SENDER_EMAIL: !Ref SenderEmail
          API_KEY: !Ref ApiKey
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - Statement:
            - Effect: Allow
              Action: ses:SendEmail
              Resource: "*"
      Events:
        Post:
          Type: HttpApi
          Properties:
            Path: /sync
            Method: POST

  AsyncOrderFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: async-order.handler
      Runtime: nodejs22.x
      CodeUri: src/
      MemorySize: 256
      Timeout: 15
      Environment:
        Variables:
          QUEUE_URL: !Ref OrdersQueue
          API_KEY: !Ref ApiKey
      Policies:
        - SQSSendMessagePolicy:
            QueueName: !GetAtt OrdersQueue.QueueName
      Events:
        Post:
          Type: HttpApi
          Properties:
            Path: /async
            Method: POST
```
*d) Add an output* to the `Outputs` section:
```yaml
  ApiUrl:
    Value: !Sub "https://${ServerlessHttpApi}.execute-api.${AWS::Region}.amazonaws.com"
```
#### 2. Write the code

*src/sync-order.mjs* (the baseline: no retries, no duplicate protection):
```javascript
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
```

*src/async-order.mjs* (the proposed approach: accept the order and reply immediately):
```javascript
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
```
**Change one line in `src/index.mjs`** so the completion time is measured from
the moment the API accepts the order:
```javascript
// before
Item: { orderId: order.orderId, status: "RECEIVED", createdAt: Date.now() },
// after
Item: { orderId: order.orderId, status: "RECEIVED", createdAt: order.acceptedAt ?? Date.now() },
```
#### 3. Deploy

Generate a secret key and redeploy (this time you need `--guided` because there
is a new parameter):
```powershell
[guid]::NewGuid().ToString("N")     # copy this string to use as ApiKey
sam validate
sam build
sam deploy --guided
```
Keep the previous values and enter `ApiKey` when asked. Get the shared variables:
```powershell
$api   = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text
$table = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue" --output text
$sm    = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='StateMachineArn'].OutputValue" --output text
$env:API_KEY = "<the-ApiKey-you-just-created>"
$env:AWS_REGION = "ap-southeast-1"
```

>Do not push `ApiKey` or `samconfig.toml` to GitHub. These two API endpoints are
>public on the Internet (protected only by a key), so they will be deleted in
>section 5.8.


#### 4. Prepare the measurement tools

In the `order-system` directory:
```powershell
mkdir tools
cd tools
npm init -y
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```
*tools/loadtest.mjs* sends N orders with C concurrent requests and measures the response time:
```javascript
// Usage: node loadtest.mjs <url> <prefix> <orders> <concurrency>
const [url, prefix, n, c] = process.argv.slice(2);
const N = Number(n), C = Number(c);
const key = process.env.API_KEY;
const email = process.env.CUSTOMER_EMAIL ?? "success@simulator.amazonses.com";
if (!url || !key) { console.error("Missing url or API_KEY"); process.exit(1); }

const lat = [];
let ok = 0, fail = 0, next = 1;

async function worker() {
  while (true) {
    const i = next++;
    if (i > N) return;
    const body = JSON.stringify({ orderId: `${prefix}-${i}`, customerEmail: email });
    const t0 = performance.now();
    try {
      const r = await fetch(url, {
        method: "POST",
        headers: { "content-type": "application/json", "x-api-key": key },
        body,
      });
      await r.text();
      lat.push(performance.now() - t0);
      r.ok ? ok++ : fail++;
    } catch { fail++; }
  }
}

await Promise.all(Array.from({ length: C }, worker));
lat.sort((a, b) => a - b);
const avg = lat.length ? lat.reduce((s, x) => s + x, 0) / lat.length : NaN;
const p95 = lat.length ? lat[Math.min(lat.length - 1, Math.ceil(lat.length * 0.95) - 1)] : NaN;
console.log(`${url} | sent=${N} concurrency=${C} | success=${ok} errors=${fail} | API avg=${avg.toFixed(0)}ms p95=${p95.toFixed(0)}ms`);
```
*tools/report.mjs* reads DynamoDB to summarize the processing results by prefix:
```javascript
// Usage: node report.mjs <table-name> <prefix> <orders-sent>
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, ScanCommand } from "@aws-sdk/lib-dynamodb";

const [table, prefix, n] = process.argv.slice(2);
const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));

const items = [];
let key;
do {
  const r = await db.send(new ScanCommand({
    TableName: table,
    ConsistentRead: true,
    FilterExpression: "begins_with(orderId, :p)",
    ExpressionAttributeValues: { ":p": `${prefix}-` },
    ExclusiveStartKey: key,
  }));
  items.push(...(r.Items ?? []));
  key = r.LastEvaluatedKey;
} while (key);

const status = {};
for (const i of items) status[i.status] = (status[i.status] ?? 0) + 1;
const extra = items.length && items.every(i => Number.isFinite(i.emailCount)) ? items.reduce((s, i) => s + Math.max(0, i.emailCount - 1), 0) : "N/A";
const d = items.filter(i => i.doneAt && i.createdAt)
  .map(i => Date.parse(i.doneAt) - i.createdAt).sort((a, b) => a - b);
const avg = d.length ? Math.round(d.reduce((s, x) => s + x, 0) / d.length) : null;
const p95 = d.length ? d[Math.min(d.length - 1, Math.ceil(d.length * 0.95) - 1)] : null;

console.log(`Prefix: ${prefix} | sent: ${n} | found: ${items.length} | lost: ${Number(n) - items.length}`);
console.log("Status:", status);
console.log("Extra emails (duplicates):", extra);
console.log(`Completion time (ms): avg=${avg} p95=${p95} (orders: ${d.length})`);
if (items.length <= 10)
  console.table(items.map(i => ({ orderId: i.orderId, status: i.status, emailCount: i.emailCount ?? "N/A" })));
```
## Experiment 1: latency and load

### Reference procedure
These commands use the example tools in section 4, run from `tools`. The screenshots use a different tool version with different arguments and output formatting; do not mix the versions. Reference commands are not evidence that every step was performed in the captured run.

```powershell
$env:CUSTOMER_EMAIL = "success@simulator.amazonses.com"
node loadtest.mjs "$api/sync"  warm-s 3 1
node loadtest.mjs "$api/async" warm-a 3 1
Start-Sleep -Seconds 60

node loadtest.mjs "$api/sync" s20 20 10
Start-Sleep -Seconds 60
node loadtest.mjs "$api/async" a20 20 10
Start-Sleep -Seconds 120

node report.mjs $table s20 20
node report.mjs $table a20 20
```

### Recorded evidence
Request `smoke-a9` returns HTTP `202` and `ACCEPTED`. This confirms request acceptance, not completed processing.

![API accepts the request](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-api-202.png)

Run `test-async-1794473095218` uses concurrency 1 and records 20/20 successful requests, averaging **135.30 ms**, with P95 **245.45 ms**. The DynamoDB report for the same prefix finds 20 `DONE` orders and no `FAILED` or other statuses.

![API results for 20 requests](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-20-loadtest.png)
![DynamoDB reports 20 DONE orders](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-20-dynamodb.png)

Run `test-async-1794473291865` records 50/50 successful requests, averaging **102.47 ms**, with P95 **166.29 ms**. The matching report finds 50 `DONE` orders. Its launch command is not shown, so concurrency is not verified.

![API and DynamoDB results for 50 orders](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-50-results.png)

Sample execution `test-async-1794471082647-2` shows successful `SendCustomerEmail` and `MarkDone` tasks and completes in approximately **590 ms**. It belongs to a different run and is not used to calculate averages for the two groups above. A successful SES task does not establish inbox delivery.

![Successful sample execution](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-execution-sample.png)

## Experiment 2: duplicate orders

### Reference procedure
Run from `tools` using the example version in section 4. The `dup-s`/`dup-a` IDs below differ from `duplicate-h7-001` in the screenshots.

```powershell
$h = @{ "x-api-key" = $env:API_KEY }
$body = '{"orderId":"dup-s","customerEmail":"success@simulator.amazonses.com"}'
1..5 | ForEach-Object {
  Invoke-RestMethod -Method Post -Uri "$api/sync" -Headers $h -ContentType "application/json" -Body $body
  Start-Sleep -Seconds 1
}

$body = '{"orderId":"dup-a","customerEmail":"success@simulator.amazonses.com"}'
1..5 | ForEach-Object {
  Invoke-RestMethod -Method Post -Uri "$api/async" -Headers $h -ContentType "application/json" -Body $body
  Start-Sleep -Seconds 1
}

Start-Sleep -Seconds 40
node report.mjs $table dup 2
```

### Recorded evidence
Execution `duplicate-h7-001` contains one successful `SendCustomerEmail` task and one successful `MarkDone` task and finishes in approximately **325 ms**. The DynamoDB report finds one `DONE` record for the same ID.

![Execution history for duplicate-h7-001](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/duplicate-h7-history.png)
![Record for duplicate-h7-001](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/duplicate-h7-dynamodb.png)

**Limits:** the screenshots do not show five submissions of the same ID or logs skipping duplicate requests. They establish only the displayed execution and record, not system-wide processing counts or exactly-once email sending. `emailCount = N/A` must not be interpreted as 0 or 1.

## Experiment 3: failures and alerts

### Reference failure procedure
These commands use the example version in section 4, from `tools`. An unverified recipient can be tested in the SES sandbox. Inspect actual execution status instead of relying only on a fixed wait.

```powershell
$env:CUSTOMER_EMAIL = "unverified@example.com"
node loadtest.mjs "$api/sync"  f-s 5 1
node loadtest.mjs "$api/async" f-a 5 1
```

```powershell
node report.mjs $table f-s 5
node report.mjs $table f-a 5
```

### Failure evidence
Execution `failure-h8-001` ends as `Failed`, with `OrderProcessingFailed` at `OrderFailed`, after **14.994 seconds**. The detailed cause is collapsed, so SES email rejection is not established as the root cause. The DynamoDB report for the same ID records `DONE = 0`, `FAILED = 1`.

![Failed execution failure-h8-001](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/failure-h8-execution.png)
![Order failure-h8-001 is FAILED](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/failure-h8-dynamodb.png)

### Alarm history
Reference commands for execution end timestamps and alarm history:

```powershell
aws stepfunctions list-executions --state-machine-arn $sm --status-filter FAILED --max-results 5 --query "executions[].[name,startDate,stopDate]" --output table
```

```powershell
aws cloudwatch describe-alarm-history --alarm-name order-sfn-failed --history-item-type StateUpdate --max-items 5 --query "AlarmHistoryItems[].[Timestamp,HistorySummary]" --output table
```

The CLI screenshot shows an execution end time of **November 12, 2026, 15:54:41.984** and an `OK` to `ALARM` transition at **15:55:06.592**, both UTC+07:00. The difference is **24.608 seconds**, rounded to **24.61 seconds**. This is not email delivery latency; check the alarm dimensions to confirm the monitored state machine.

![Execution end time and alarm history](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/failure-h8-alarm-history.png)

### DLQ check — no evidence in this set
Run this reference procedure only while the `order-system` stack exists. A Step Functions alarm screenshot is not evidence of a DLQ alarm. This set also does not establish receipt of an admin alert email.

```powershell
(Get-Date).ToUniversalTime().ToString("o")
aws sqs send-message --queue-url (aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue" --output text) --message-body "not-json"
aws cloudwatch describe-alarm-history --alarm-name order-dlq-has-messages --history-item-type StateUpdate --max-items 3 --query "AlarmHistoryItems[].[Timestamp,HistorySummary]" --output table
```

## Summary of Experimental Results

| Metric | Asynchronous — 20 orders | Asynchronous — 50 orders |
|---|---:|---:|
| Concurrency verified from screenshot | 1 | Not verified |
| Successful API requests | 20/20 | 50/50 |
| Average API response time (ms) | 135.30 | 102.47 |
| P95 API response time (ms) | 245.45 | 166.29 |
| Records found / DONE | 20 / 20 | 50 / 50 |
| FAILED / other statuses | 0 / 0 | 0 / 0 |
| Emails sent | Not verified | Not verified |

Synchronous benchmark results and average Step Functions execution times for these groups are not evidenced and are excluded. API response time is not end-to-end processing time. These results describe the captured runs only and do not establish superiority over synchronous processing.

## Cost Evaluation
Cost Explorer shows **Total costs = USD 0.00** in the `Nov-12*` column; visible service rows include Glue, SQS, and CloudWatch. Full filters and the service list are not shown, so the screenshot does not establish that the whole workshop incurred no cost. Recheck the query period, filters, and refreshed data.

![Cost Explorer snapshot](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/cost-explorer-snapshot.png)

## Conclusion
The documented asynchronous runs show all 20 and 50 orders as `DONE`. Failure evidence includes a failed execution, a `FAILED` order, and an alarm transition history. Duplicate evidence establishes only the execution and record for `duplicate-h7-001`, not exactly-once processing or email sending. Cost and end-to-end performance comparisons require further data.