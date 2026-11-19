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
![Deploy result](/images/5-Workshop/5.7-Experiments/deploy-result.png)

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
const extra = items.reduce((s, i) => s + Math.max(0, (i.emailCount ?? 0) - 1), 0);
const d = items.filter(i => i.doneAt && i.createdAt)
  .map(i => Date.parse(i.doneAt) - i.createdAt).sort((a, b) => a - b);
const avg = d.length ? Math.round(d.reduce((s, x) => s + x, 0) / d.length) : null;
const p95 = d.length ? d[Math.min(d.length - 1, Math.ceil(d.length * 0.95) - 1)] : null;

console.log(`Prefix: ${prefix} | sent: ${n} | found: ${items.length} | lost: ${Number(n) - items.length}`);
console.log("Status:", status);
console.log("Extra emails (duplicates):", extra);
console.log(`Completion time (ms): avg=${avg} p95=${p95} (orders: ${d.length})`);
if (items.length <= 10)
  console.table(items.map(i => ({ orderId: i.orderId, status: i.status, emailCount: i.emailCount ?? 0 })));
```
## Experiment 1: latency and load

Run a warm-up first (so Lambda cold starts are not counted), then measure
20 orders. Wait between runs so SES can "cool down" (limit of 1 email/second):
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

![Loadtest result](/images/5-Workshop/5.7-Experiments/loadtest-result.png)

![Report result](/images/5-Workshop/5.7-Experiments/report-result.png)

Repeat with 50 orders (`s50`, `a50`, wait about 4 minutes before running `report`).
Run each load level at least 3 times and take the average.

![loadtest result](/images/5-Workshop/5.7-Experiments/50.png)

What to read from the output:

- `success` and `errors` for each API: does the synchronous path fail when SES rate-limits?
- `Status` in the report: how many orders are `DONE`, `FAILED`, `RECEIVED` (stuck), and how many are `lost`.
- `API avg/p95` (what the customer sees) versus `Completion time` (when processing finishes).

## Experiment 2: duplicate orders

Send the same `orderId` 5 times to each path:
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
node ..\tools\report.mjs $table dup 2
```
(If you are already in the `tools` directory, use `node report.mjs $table dup 2`.)
The result table shows the `emailCount` of each order. By design, the
synchronous path sends an email every time it receives a request, while the
asynchronous path only processes the first order.

![Duplicate result](/images/5-Workshop/5.7-Experiments/duplicate-result.png)

## Experiment 3: failures and alerts

Use an unverified email address so SES rejects it, and send 5 orders to each path:
```powershell
$env:CUSTOMER_EMAIL = "unverified@example.com"
node loadtest.mjs "$api/sync"  f-s 5 1
node loadtest.mjs "$api/async" f-a 5 1
```
Wait about 3 minutes for the executions to exhaust their retries and fail, then:
```powershell
node report.mjs $table f-s 5
node report.mjs $table f-a 5
```
Observe:

- Synchronous path: the customer gets a 500 error immediately, the order stays in `RECEIVED`, and *no alert is raised* (the baseline is intentionally kept simple, with no monitoring).
- Asynchronous path: the API still returns 202, the order is marked `FAILED`, the admin receives an alert email, and the CloudWatch alarm fires.

*Measure the time to alert.* The timestamp of the direct alert (SNS) is the
moment the execution ends:
```powershell
aws stepfunctions list-executions --state-machine-arn $sm --status-filter FAILED --max-results 5 --query "executions[].[name,startDate,stopDate]" --output table
```
The timestamp of the CloudWatch alarm:
```powershell
aws cloudwatch describe-alarm-history --alarm-name order-sfn-failed --history-item-type StateUpdate --max-items 5 --query "AlarmHistoryItems[].[Timestamp,HistorySummary]" --output table
```
Take the difference between `stopDate` and `Timestamp` to see how much later the
alarm fires than the direct alert (note that the two tables may display
different time zones).

*Measure the DLQ path* (malformed messages): record the time of sending, then
compare it with the history of the `order-dlq-has-messages` alarm:
```powershell
(Get-Date).ToUniversalTime().ToString("o")
aws sqs send-message --queue-url (aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue" --output text) --message-body "not-json"
aws cloudwatch describe-alarm-history --alarm-name order-dlq-has-messages --history-item-type StateUpdate --max-items 3 --query "AlarmHistoryItems[].[Timestamp,HistorySummary]" --output table
```
![Failed execution](/images/5-Workshop/5.7-Experiments/failed-execution.png)
![Alarm history](/images/5-Workshop/5.7-Experiments/alarm-history.png)


## Summary of Experimental Results

The table below summarizes the actual test results of the synchronous and asynchronous order processing approaches on AWS.

| Metric | Synchronous - 20 orders | Asynchronous - 20 orders | Asynchronous - 50 orders |
|---|---:|---:|---:|
| Concurrency | 5 | 1 | 1 |
| Successful API requests | 19/20 | 20/20 | 50/50 |
| Success rate | 95% | 100% | 100% |
| Average API response time (ms) | 774.74 | 135.30 | 102.47 |
| P95 API response time (ms) | 2,238.70 | 245.45 | 166.29 |
| Completed orders (DONE) | 19 | 20 | 50 |
| Failed orders (FAILED) | To be verified | 0 | 0 |
| Orders with no record | 1 | 0 | 0 |

### Additional Test Results

| Metric | Result |
|---|---|
| Average Step Functions execution time (20 orders) | 419.85 ms |
| Average Step Functions execution time (50 orders) | 334.70 ms |
| Duplicate order test | Same order ID submitted 5 times |
| Emails successfully sent via SES | 1 |
| CloudWatch alarm | Triggered successfully |
| Alarm trigger time | 24.61 seconds |

### Performance Analysis

Based on the experimental results, the asynchronous approach shows an advantage in API response time over the synchronous approach.

For 20 orders, the synchronous approach achieved a 95% success rate, with an average API response time of 774.74 ms and a P95 of 2,238.70 ms.

In contrast, the asynchronous approach achieved a 100% success rate, with an average API response time of 135.30 ms and a P95 of 245.45 ms.

When the load was increased to 50 orders, the asynchronous approach still maintained a 100% success rate. The average API response time was 102.47 ms, and the P95 was 166.29 ms.

These results show that the asynchronous approach can accept requests quickly and remain stable in the test scenarios that were run.

However, the concurrency level was not the same across the test runs. Therefore, there is not enough evidence to conclude that the asynchronous approach has better end-to-end processing performance than the synchronous approach.

### Observations and Limitations

- For the synchronous approach with 20 orders, 19 requests succeeded and 1 request returned an HTTP 500 error and was not recorded in DynamoDB.
- For the asynchronous approach, both the 20-order and 50-order loads achieved a 100% completion rate.
- The synchronous approach was tested with a concurrency of 5, while the asynchronous approach was tested with a concurrency of 1. This difference may affect the performance comparison.
- API response time does not equal the total time to complete the order processing workflow, especially in an asynchronous architecture.
- The number of FAILED orders for the synchronous approach in DynamoDB needs to be verified before finalizing the report.
- The results are for reference only and apply to the environment and the number of tests that were run.


## Evaluation of Duplicate Prevention and Failure Handling

| Scenario | Result |
|---|---|
| Same `orderId` sent 5 times (asynchronous) | Only 1 SES email sent successfully, `emailCount` = 1 |
| Same `orderId` sent 5 times (synchronous) | `emailCount` = ... (fill in your measured value) |
| Failed order (unverified email) | Order marked `FAILED`, CloudWatch alarm changed to **In alarm** after about **24.61 seconds** |

These results show that duplicate prevention (DynamoDB and Step Functions) and failure recording and alerting work as designed. State clearly what the 24.61-second figure is measured from (for example, from the `stopDate` of the failed execution to the alarm `Timestamp`).

## Cost Evaluation

Costs were tracked with AWS Cost Explorer, grouped by service. At the time of checking (November 12, 2026), the total cost was approximately **USD 0.00**. Lambda, Step Functions, and DynamoDB showed no significant charges.

Cost Explorer data is updated with a delay, so the actual cost should be re-checked after 24 to 48 hours. The test volume was small (a few dozen orders), so the cost is expected to stay within the free tier and below the USD 5 budget set in section 5.2.

## Conclusion

The asynchronous order processing system on AWS runs reliably in the scenarios tested:

- **API response time:** 135.30 ms (20 orders) and 102.47 ms (50 orders, p95 = 166.29 ms), lower than the synchronous approach (774.74 ms).
- **Reliability:** 100% of asynchronous orders reached `DONE`, while the synchronous approach reached 95% at 20 orders.
- **Duplicate prevention and monitoring:** sending the same order 5 times resulted in only one processing run, and a failed order was marked `FAILED` with an alert after about 24.61 seconds.

**Limitations:** each load level was run only a few times, the two approaches used different concurrency levels, the SES sandbox limits sending to 1 email per second, and the final cost has not yet been confirmed. The results are therefore indicative only. For firmer conclusions about end-to-end performance, repeat the tests with the same concurrency level and run each several times.