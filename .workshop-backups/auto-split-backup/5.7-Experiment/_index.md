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


<!-- evidence-deployment:start -->

**Supplementary deployment:** order-api-evidence reuses order-orchestration-evidence resources. The evidence below is from November 13, 2026.

![Deployment of comparison APIs](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-api-deploy-success.png)

<!-- evidence-deployment:end -->

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

On **November 13, 2026**, two APIs in **order-api-evidence** were tested using the DynamoDB table and workflow in **order-orchestration-evidence**. Run ID: **cmp-20261113160505330-eac6bb**.

Each mode used **20 measured requests**, concurrency **1**, a **1,500 ms pause after each response**, and two excluded warm-up requests. Synchronous ran first, then asynchronous. Recipient: success@simulator.amazonses.com. The actual tool was workshop-benchmark-tools/compare.cjs; illustrative tools above may use a different output format.

### Synchronous results

![Synchronous results for 20 orders](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-sync-20.png)

### Asynchronous results

![Asynchronous results for 20 orders](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-async-20.png)

### Comparison

| Metric | Synchronous | Asynchronous |
|---|---:|---:|
| Measured requests / concurrency | 20 / 1 | 20 / 1 |
| Pause after response (ms) | 1500 | 1500 |
| Successful HTTP requests | 20/20 | 20/20 |
| HTTP / network errors | 0 / 0 | 0 / 0 |
| Average API latency (ms) | 229.55 | 109.28 |
| P95 API latency (ms) | 343.33 | 110.62 |
| Records found / DONE | 20 / 20 | 20 / 20 |
| FAILED / other / not observed | 0 / 0 / 0 | 0 / 0 / 0 |
| Completion timestamp samples | 20 | 20 |
| Average approximate completion (ms) | 122.75 | 445.55 |
| P95 approximate completion (ms) | 138 | 516 |

![Comparison with matching test settings](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-summary.png)

**Observation:** the asynchronous API responded faster in this run, while approximate completion took longer. HTTP 202 confirms acceptance; DONE is checked separately in DynamoDB. Client API latency excludes the pacing pause. Approximate completion uses application createdAt and doneAt, with different measurement boundaries; it is neither inbox delivery time nor exact Step Functions duration.

This is one small paced test per mode, not a maximum-throughput measurement. Equal pauses after responses do not ensure equal arrival rates. Downloads/workshop-api-evidence.json contains an API key and must not be published.

## Experiment 2: duplicate orders

Identical messages for **dup-proof-20261113-220937-a7468d** were submitted to SQS **five times**. Results: execution **SUCCEEDED**, **one** successful SES task in that execution, DynamoDB **DONE**, and **emailCount = 1**.

![Duplicate-test validation](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-validation.png)

Logs show duplicates being skipped. Two workflow-started log lines do not establish two executions: Standard StartExecution with the same name and input can return an existing running execution.

![Logs showing duplicate orders skipped](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-skip-logs.png)

The inbox received a confirmation with the matching order ID. These observations support duplicate handling in this test, not guaranteed exactly-once email delivery under all failures.

![Received confirmation email](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-confirmation-email.png)

## Experiment 3: failures and alerts

### SES failure and retries

Order **failure-proof-20261113-222244-4343a2** used **not-an-email**. SES returned **SesV2.BadRequestException**, HTTP **400**, cause **Missing final '@domain'**. Event history shows task rescheduling and execution after failures. SDK Attempt Count: 1 is not the total Step Functions retry count.

![SES error cause and retry history](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-ses-error-history.png)

### Failure branch

Catch routed to **MarkFailed** and **NotifyAdminFailure**, both successful. The workflow terminated at **OrderFailed**; MarkDone was skipped.

![Workflow failure-handling branch](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-workflow-graph.png)

The SNS email contains the matching order ID and was sent by NotifyAdminFailure.

![Alert email from the workflow](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-admin-email.png)

### DLQ and CloudWatch Alarm

A separate test used invalid JSON: **not-json-dlq-proof-20261113-223327-eed369**. Inspect the message in **order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU**; the screenshot should show both the queue name and message body.

![Malformed message in the identified DLQ](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-malformed-message.png)

Alarm **order-orchestration-evidence-dlq-proof** monitored the DLQ ApproximateNumberOfMessagesVisible metric and entered **ALARM** at **November 13, 2026, 15:39:57 UTC**.

![Triggered DLQ alarm](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-cloudwatch-alarm.png)

The CloudWatch email delivered through SNS records the same alarm and transition time, with datapoint **1** exceeding threshold **0**. This is distinct from the workflow notification above. These screenshots do not measure alert latency.

![CloudWatch alarm email via SNS](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-alarm-email.png)

## Cost Evaluation

Complete cost data for the new API stack and alarm from November 13, 2026 is unavailable; total cost is not claimed to be zero.

## Conclusion

Both modes reached DONE for 20/20 orders in this run. Separate tests recorded duplicate handling, SES retries, SNS failure notifications and DLQ alerts. Findings are limited to the documented test conditions and runs.