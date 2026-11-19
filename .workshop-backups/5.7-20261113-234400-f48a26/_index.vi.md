---
title: "Thực nghiệm lỗi và tải"
weight: 7
chapter: false
pre: " <b> 5.7 </b> "
---
## Mục tiêu

So sánh hai cách xử lý đơn hàng trong cùng điều kiện:

| | Đồng bộ (đối chứng) | Bất đồng bộ (đề xuất) |
|---|---|---|
| Luồng | API ghi DynamoDB rồi gửi SES ngay trong request | API chỉ đưa đơn vào SQS; Lambda và Step Functions xử lý sau |
| Chống trùng | Không | Có (DynamoDB) |
| Thử lại, DLQ, cảnh báo | Không | Có |

Chỉ số: phản hồi API (trung bình, p95), đơn hoàn tất, đơn thất bại hoặc mất,
thời gian hoàn tất xử lý, số email dư khi gửi trùng, và thời gian có cảnh báo
khi lỗi.

>**Giới hạn của SES sandbox:** tối đa 1 email/giây. Email gửi đến
>`success@simulator.amazonses.com` (hộp thư giả lập của SES) không tính vào
>hạn mức 200 email/ngày nhưng vẫn bị giới hạn 1 email/giây. Vì vậy mức tải thử
>là 20 và 50 đơn. Đây cũng là một kết quả đáng ghi nhận: nút thắt nằm ở dịch
>vụ gửi email chứ không phải ở kiến trúc.


#### 1. Cập nhật template.yaml

*a) Thêm tham số* vào phần Parameters:
```yaml
  ApiKey:
    Type: String
    NoEcho: true
    MinLength: 16
    Description: Khoa bi mat cho header x-api-key
```
*b) Sửa state machine.* Trong SendCustomerEmail, thay khối Retry để chịu
được việc SES giới hạn tốc độ:
```yaml
            Retry:
              - ErrorEquals: [States.ALL]
                IntervalSeconds: 2
                MaxAttempts: 6
                BackoffRate: 2
                JitterStrategy: FULL
```
Thay toàn bộ trạng thái `MarkDone` để ghi thêm thời điểm hoàn tất và đếm số
email đã gửi:
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
>Với `MaxAttempts: 6` và hệ số 2, một đơn bị lỗi vĩnh viễn (ví dụ email chưa
>xác minh) sẽ mất tối đa khoảng 2 phút mới được đánh dấu `FAILED`, thay vì
>20 đến 30 giây như ở mục 5.5.


*c) Thêm hai hàm API* vào cuối `Resources` (trước `Outputs`:):
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
*d) Thêm đầu ra* vào phần `Outputs`:
```yaml
  ApiUrl:
    Value: !Sub "https://${ServerlessHttpApi}.execute-api.${AWS::Region}.amazonaws.com"
```
#### 2. Viết mã

*src/sync-order.mjs* (phương án đối chứng: không thử lại, không chống trùng):
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
        Subject: { Data: `Xac nhan don hang ${order.orderId}` },
        Body: { Text: { Data: `Cam on ban. Don hang ${order.orderId} da duoc ghi nhan.` } },
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
    console.error("Loi xu ly don", order.orderId, e.name);
    return { statusCode: 500, body: "error" };
  }
};
```

*src/async-order.mjs* (phương án đề xuất: nhận đơn rồi trả lời ngay):
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
**Sửa một dòng trong `src/index.mjs`** để thời gian hoàn tất được tính từ lúc
API nhận đơn:
```javascript
// truoc
Item: { orderId: order.orderId, status: "RECEIVED", createdAt: Date.now() },
// sau
Item: { orderId: order.orderId, status: "RECEIVED", createdAt: order.acceptedAt ?? Date.now() },
```
#### 3. Triển khai

Tạo một khóa bí mật và triển khai lại (lần này cần `--guided` vì có tham số mới):
```powershell
[guid]::NewGuid().ToString("N")     # copy chuoi nay lam ApiKey
sam validate
sam build
sam deploy --guided
```
Giữ nguyên các giá trị cũ, nhập `ApiKey` khi được hỏi. Lấy các biến dùng chung:
```powershell
$api   = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text
$table = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue" --output text
$sm    = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='StateMachineArn'].OutputValue" --output text
$env:API_KEY = "<ApiKey-ban-vua-tao>"
$env:AWS_REGION = "ap-southeast-1"
```

>Không đưa `ApiKey` và `samconfig.toml` lên GitHub. Hai đường API này công
>khai trên Internet (chỉ được bảo vệ bằng khóa), nên sẽ bị xóa ở mục 5.8.


#### 4. Chuẩn bị công cụ đo

Trong thư mục `order-system`:
```powershell
mkdir tools
cd tools
npm init -y
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```
*tools/loadtest.mjs* gửi N đơn với C yêu cầu đồng thời và đo thời gian phản hồi:
```javascript
// Cach dung: node loadtest.mjs <url> <tien-to> <so-don> <dong-thoi>
const [url, prefix, n, c] = process.argv.slice(2);
const N = Number(n), C = Number(c);
const key = process.env.API_KEY;
const email = process.env.CUSTOMER_EMAIL ?? "success@simulator.amazonses.com";
if (!url || !key) { console.error("Thieu url hoac API_KEY"); process.exit(1); }

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
console.log(`${url} | gui=${N} dong thoi=${C} | thanh cong=${ok} loi=${fail} | API avg=${avg.toFixed(0)}ms p95=${p95.toFixed(0)}ms`);
```
*tools/report.mjs* đọc DynamoDB để tổng hợp kết quả xử lý theo tiền tố:
```javascript
// Cach dung: node report.mjs <ten-bang> <tien-to> <so-don-da-gui>
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

console.log(`Tien to: ${prefix} | gui: ${n} | tim thay: ${items.length} | mat: ${Number(n) - items.length}`);
console.log("Trang thai:", status);
console.log("Email du (gui trung):", extra);
console.log(`Thoi gian hoan tat (ms): avg=${avg} p95=${p95} (so don: ${d.length})`);
if (items.length <= 10)
  console.table(items.map(i => ({ orderId: i.orderId, status: i.status, emailCount: i.emailCount ?? "N/A" })));
```
## Thí nghiệm 1: độ trễ và tải

### Quy trình tham khảo
Các lệnh dưới đây sử dụng phiên bản công cụ hướng dẫn ở mục 4, chạy trong thư mục `tools`. Ảnh chụp sử dụng phiên bản công cụ khác về tham số và định dạng đầu ra; không dùng lẫn hai phiên bản. Các lệnh tham khảo không phải bằng chứng rằng tất cả bước đã được thực hiện trong lần chạy trên ảnh.

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

Yêu cầu `smoke-a9` nhận HTTP `202`, trạng thái `ACCEPTED`. Đây là xác nhận tiếp nhận yêu cầu, chưa phải xác nhận hoàn tất xử lý.

![API tiếp nhận yêu cầu](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-api-202.png)

Lần chạy `test-async-1794473095218` có mức đồng thời 1, thành công 20/20 yêu cầu; trung bình **135,30 ms**, P95 **245,45 ms**. Báo cáo DynamoDB cùng tiền tố tìm thấy 20 đơn `DONE`, không có `FAILED` hoặc trạng thái khác.

![Kết quả API với 20 yêu cầu](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-20-loadtest.png)
![DynamoDB ghi nhận 20 đơn DONE](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-20-dynamodb.png)

Lần chạy `test-async-1794473291865` có 50/50 yêu cầu thành công; trung bình **102,47 ms**, P95 **166,29 ms**. Báo cáo cùng tiền tố ghi nhận 50 đơn `DONE`. Ảnh không hiển thị lệnh khởi chạy nên chưa xác nhận mức đồng thời.

![Kết quả API và DynamoDB với 50 đơn](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-50-results.png)

Execution mẫu `test-async-1794471082647-2` có `SendCustomerEmail` và `MarkDone` thành công, kết thúc sau khoảng **590 ms**. Mẫu này thuộc lần chạy khác; không dùng để tính trung bình của hai nhóm trên. Tác vụ SES thành công không chứng minh email đã đến hộp thư người nhận.

![Execution mẫu thành công](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/async-execution-sample.png)

## Thí nghiệm 2: gửi trùng đơn

### Quy trình tham khảo
Chạy từ `tools` với phiên bản công cụ ở mục 4. Các mã `dup-s`/`dup-a` trong lệnh khác mã `duplicate-h7-001` trên ảnh.

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

Execution `duplicate-h7-001` có một tác vụ `SendCustomerEmail` thành công, một tác vụ `MarkDone` thành công và kết thúc sau khoảng **325 ms**. Báo cáo DynamoDB tìm thấy một bản ghi cùng mã đơn ở trạng thái `DONE`.

![Lịch sử execution duplicate-h7-001](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/duplicate-h7-history.png)
![Bản ghi duplicate-h7-001](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/duplicate-h7-dynamodb.png)

**Giới hạn:** chưa có ảnh gửi cùng mã đơn 5 lần hoặc log bỏ qua đơn trùng. Các ảnh chỉ xác nhận execution và bản ghi đang hiển thị, chưa đủ chứng minh số lần xử lý trên toàn hệ thống hoặc gửi email đúng một lần. `emailCount = N/A` không được coi là 0 hoặc 1.

## Thí nghiệm 3: lỗi và cảnh báo

### Quy trình tạo lỗi tham khảo
Các lệnh sau dùng phiên bản công cụ ở mục 4, chạy từ `tools`. Trong SES sandbox, có thể thử địa chỉ người nhận chưa xác minh. Cần kiểm tra trạng thái execution thực tế, không chỉ dựa vào thời gian chờ cố định.

```powershell
$env:CUSTOMER_EMAIL = "chua-xac-minh@example.com"
node loadtest.mjs "$api/sync"  f-s 5 1
node loadtest.mjs "$api/async" f-a 5 1
```

```powershell
node report.mjs $table f-s 5
node report.mjs $table f-a 5
```

Execution `failure-h8-001` kết thúc ở `Failed`, lỗi `OrderProcessingFailed` tại bước `OrderFailed`, sau **14,994 giây**. Ảnh chưa mở nguyên nhân chi tiết nên không khẳng định nguyên nhân gốc là SES từ chối email. Báo cáo DynamoDB cùng mã đơn ghi nhận `DONE = 0`, `FAILED = 1`.

![Execution failure-h8-001 thất bại](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/failure-h8-execution.png)
![Đơn failure-h8-001 có trạng thái FAILED](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/failure-h8-dynamodb.png)

### Lịch sử cảnh báo
Các lệnh tham khảo để lấy mốc kết thúc execution và lịch sử alarm:

```powershell
aws stepfunctions list-executions --state-machine-arn $sm --status-filter FAILED --max-results 5 --query "executions[].[name,startDate,stopDate]" --output table
```

```powershell
aws cloudwatch describe-alarm-history --alarm-name order-sfn-failed --history-item-type StateUpdate --max-items 5 --query "AlarmHistoryItems[].[Timestamp,HistorySummary]" --output table
```

Ảnh CLI hiển thị execution kết thúc lúc **12/11/2026 15:54:41.984**, alarm chuyển `OK` sang `ALARM` lúc **15:55:06.592**, cùng múi giờ UTC+07:00. Chênh lệch là **24,608 giây**, làm tròn **24,61 giây**. Đây không phải thời gian email cảnh báo đến hộp thư; cần đối chiếu dimension của alarm để xác nhận đúng state machine được giám sát.

![Mốc kết thúc execution và lịch sử alarm](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/failure-h8-alarm-history.png)

### Kiểm tra DLQ — chưa có minh chứng trong bộ ảnh này
Quy trình tham khảo sau chỉ chạy khi stack `order-system` còn tồn tại. Không dùng ảnh alarm Step Functions để minh chứng alarm DLQ. Bộ ảnh cũng chưa chứng minh admin đã nhận email cảnh báo.

```powershell
(Get-Date).ToUniversalTime().ToString("o")
aws sqs send-message --queue-url (aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue" --output text) --message-body "khong-phai-json"
aws cloudwatch describe-alarm-history --alarm-name order-dlq-has-messages --history-item-type StateUpdate --max-items 3 --query "AlarmHistoryItems[].[Timestamp,HistorySummary]" --output table
```

## Tổng hợp kết quả thực nghiệm

| Chỉ số | Bất đồng bộ — 20 đơn | Bất đồng bộ — 50 đơn |
|---|---:|---:|
| Mức đồng thời được ảnh xác nhận | 1 | Chưa xác nhận |
| Yêu cầu API thành công | 20/20 | 50/50 |
| Phản hồi API trung bình (ms) | 135,30 | 102,47 |
| Phản hồi API P95 (ms) | 245,45 | 166,29 |
| Số bản ghi tìm thấy / DONE | 20 / 20 | 50 / 50 |
| FAILED / trạng thái khác | 0 / 0 | 0 / 0 |
| Số email đã gửi | Chưa xác nhận | Chưa xác nhận |

Chưa có ảnh xác nhận số liệu đồng bộ hoặc thời gian Step Functions trung bình của từng nhóm nên không đưa các số liệu đó vào bảng. Thời gian phản hồi API không phải thời gian hoàn tất xử lý đầu cuối. Kết quả chỉ mô tả các lần chạy được chụp, chưa chứng minh ưu thế so với phương án đồng bộ.

## Đánh giá chi phí
Cost Explorer hiển thị **Total costs = 0,00 USD**, cột `Nov-12*`; các dòng dịch vụ nhìn thấy gồm Glue, SQS và CloudWatch. Ảnh không hiển thị đầy đủ bộ lọc hay toàn bộ dịch vụ nên chưa kết luận toàn bộ workshop không phát sinh chi phí. Cần kiểm tra lại kỳ truy vấn, bộ lọc và dữ liệu sau khi cập nhật.

![Ảnh chụp Cost Explorer](/images/5-Workshop/5.7-Experiments/evidence-20261113-215406-549e83/cost-explorer-snapshot.png)

## Kết luận
Hai lần chạy bất đồng bộ được minh chứng có toàn bộ 20 và 50 đơn ở trạng thái `DONE`. Kịch bản lỗi ghi nhận execution thất bại, đơn `FAILED` và lịch sử chuyển trạng thái alarm. Minh chứng chống trùng hiện chỉ xác nhận execution và bản ghi của `duplicate-h7-001`; chưa đủ để khẳng định xử lý hoặc gửi email đúng một lần. Chi phí và phép so sánh hiệu năng đầu cuối cần thêm dữ liệu.