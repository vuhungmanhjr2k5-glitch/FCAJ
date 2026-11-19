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


<!-- evidence-deployment:start -->

**Triển khai bổ sung:** stack order-api-evidence dùng lại tài nguyên order-orchestration-evidence. Ảnh và số liệu bên dưới thuộc đợt 13/11/2026.

![Triển khai hai API so sánh](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-api-deploy-success.png)

<!-- evidence-deployment:end -->

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

Ngày **13/11/2026**, kiểm thử hai API của stack **order-api-evidence**, dùng chung bảng DynamoDB và quy trình **order-orchestration-evidence**. Mã nhóm: **cmp-20261113160505330-eac6bb**.

Mỗi phương án đo **20 yêu cầu**, đồng thời **1**, nghỉ **1.500 ms sau mỗi phản hồi**, loại hai yêu cầu khởi động. Đồng bộ chạy trước, bất đồng bộ chạy sau. Người nhận: success@simulator.amazonses.com. Công cụ thực tế: workshop-benchmark-tools/compare.cjs; các công cụ minh họa phía trên có thể khác định dạng đầu ra.

### Kết quả đồng bộ

![Kết quả đồng bộ với 20 đơn](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-sync-20.png)

### Kết quả bất đồng bộ

![Kết quả bất đồng bộ với 20 đơn](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-async-20.png)

### So sánh

| Chỉ số | Đồng bộ | Bất đồng bộ |
|---|---:|---:|
| Yêu cầu đo / đồng thời | 20 / 1 | 20 / 1 |
| Nghỉ sau phản hồi (ms) | 1500 | 1500 |
| HTTP thành công | 20/20 | 20/20 |
| Lỗi HTTP / mạng | 0 / 0 | 0 / 0 |
| API trung bình (ms) | 229,55 | 109,28 |
| API P95 (ms) | 343,33 | 110,62 |
| Bản ghi tìm thấy / DONE | 20 / 20 | 20 / 20 |
| FAILED / khác / chưa quan sát thấy | 0 / 0 / 0 | 0 / 0 / 0 |
| Mẫu thời gian hoàn tất | 20 | 20 |
| Hoàn tất xấp xỉ trung bình (ms) | 122,75 | 445,55 |
| Hoàn tất xấp xỉ P95 (ms) | 138 | 516 |

![Bảng so sánh cùng điều kiện cấu hình](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-summary.png)

**Nhận xét:** API bất đồng bộ phản hồi nhanh hơn trong lần đo này; thời gian hoàn tất xấp xỉ dài hơn. HTTP 202 xác nhận tiếp nhận; DONE được kiểm tra riêng trong DynamoDB. Độ trễ API đo tại máy khách, không tính khoảng nghỉ. Thời gian hoàn tất dùng createdAt và doneAt của ứng dụng, có mốc đo khác API; không phải thời gian email đến hộp thư hoặc duration chính xác của Step Functions.

Đây là một lần đo với tải nhỏ cho mỗi phương án, chưa chứng minh thông lượng cực đại. Cùng khoảng nghỉ sau phản hồi không đồng nghĩa cùng tốc độ đến. File Downloads/workshop-api-evidence.json chứa API key, không đưa lên website hoặc kho mã.

## Thí nghiệm 2: gửi trùng đơn

Đơn **dup-proof-20261113-220937-a7468d** được gửi cùng nội dung **5 lần** vào SQS. Kết quả: execution **SUCCEEDED**, **1** tác vụ SES thành công trong execution, DynamoDB **DONE**, **emailCount = 1**.

![Kết quả kiểm tra gửi trùng](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-validation.png)

Log ghi nhận bỏ qua đơn trùng. Hai dòng báo khởi động không chứng minh có hai execution: Standard StartExecution cùng tên và input khi execution đang chạy có thể trả về execution hiện có.

![Log bỏ qua đơn trùng](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-skip-logs.png)

Hộp thư nhận email xác nhận đúng mã đơn. Kết quả hỗ trợ chống trùng trong lần thử này; không bảo đảm email được gửi đúng một lần trong mọi tình huống lỗi.

![Email xác nhận thực nhận](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-confirmation-email.png)

## Thí nghiệm 3: lỗi và cảnh báo

### Lỗi SES và thử lại

Đơn **failure-proof-20261113-222244-4343a2** dùng địa chỉ **not-an-email**. SES trả **SesV2.BadRequestException**, HTTP **400**, nguyên nhân **Missing final '@domain'**. Event history cho thấy tác vụ được lập lịch và thực thi lại sau lỗi. SDK Attempt Count: 1 không phải tổng số lần Step Functions thử lại.

![Nguyên nhân lỗi SES và lịch sử thử lại](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-ses-error-history.png)

### Nhánh xử lý lỗi

Catch chuyển sang **MarkFailed**, rồi **NotifyAdminFailure**; cả hai thành công. Quy trình kết thúc tại **OrderFailed**; MarkDone không chạy trên nhánh này.

![Nhánh xử lý lỗi của quy trình](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-workflow-graph.png)

Email SNS chứa đúng mã đơn, do bước NotifyAdminFailure gửi.

![Email cảnh báo từ quy trình](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-admin-email.png)

### DLQ và CloudWatch Alarm

Phép thử riêng dùng thông điệp **not-json-dlq-proof-20261113-223327-eed369**, không phải JSON hợp lệ. Kiểm tra thông điệp trong DLQ **order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU**; ảnh cần hiển thị cả tên hàng đợi và nội dung.

![Thông điệp lỗi trong đúng DLQ](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-malformed-message.png)

Alarm **order-orchestration-evidence-dlq-proof** giám sát ApproximateNumberOfMessagesVisible của DLQ và chuyển sang **ALARM** lúc **13/11/2026 15:39:57 UTC**.

![Alarm DLQ được kích hoạt](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-cloudwatch-alarm.png)

Email CloudWatch qua SNS ghi cùng tên alarm và mốc chuyển trạng thái, datapoint **1** vượt ngưỡng **0**. Đây là cảnh báo CloudWatch, khác email do workflow gửi ở trên. Bộ ảnh này chưa đo độ trễ cảnh báo.

![Email cảnh báo CloudWatch qua SNS](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-alarm-email.png)

## Đánh giá chi phí

Chưa có số liệu chi phí đầy đủ cho stack API và alarm mới trong đợt 13/11/2026, nên không kết luận tổng chi phí bằng 0.

## Kết luận

Hai phương án đều đạt DONE 20/20 trong lần đo. Các phép thử riêng ghi nhận xử lý gửi trùng, thử lại khi SES lỗi, nhánh cảnh báo SNS và cảnh báo DLQ. Kết luận giới hạn trong các điều kiện và lần thử được minh chứng.