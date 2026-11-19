---
title: "Viết mã"
weight: 2
chapter: false
pre: " <b> 5.7.2 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

## Hàm HTTP đồng bộ

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
'@
$path = Join-Path $PWD.Path 'src/sync-order.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Hàm tiếp nhận bất đồng bộ

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

## Worker giữ lại thời điểm tiếp nhận

Đây là phiên bản hoàn chỉnh của `src/index.mjs` từ 5.5.2 với thời điểm `createdAt` được cập nhật.

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
    const order = JSON.parse(record.body); // JSON sai -> loi -> retry -> DLQ

    // Buoc 1: ghi so, moi orderId chi duoc ghi mot lan
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
        console.log("Don trung, bo qua:", order.orderId);
        continue;
      }
      // Con o trang thai RECEIVED: lan truoc chua khoi dong duoc quy trinh -> thu lai
    }

    // Buoc 2: khoi dong quy trinh. Ten execution = orderId nen khong the chay trung.
    try {
      await sfn.send(new StartExecutionCommand({
        stateMachineArn: process.env.STATE_MACHINE_ARN,
        name: order.orderId,
        input: JSON.stringify(order),
      }));
      console.log("Da khoi dong quy trinh:", order.orderId);
    } catch (e) {
      if (e.name === "ExecutionAlreadyExists") {
        console.log("Quy trinh da ton tai, bo qua:", order.orderId);
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

### Thêm SDK cho hai handler HTTP

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

## Đọc phản hồi theo hợp đồng của từng endpoint

Trước khi làm việc với bảng hoặc queue, cả hai handler kiểm tra khóa gửi trong header, đọc JSON và kiểm tra mã đơn. Với mã hợp lệ, ký tự chữ, số, gạch ngang và gạch dưới dùng được làm tên execution. Đường thành công của handler đồng bộ trả 200 sau khi các thao tác trong request hoàn tất. Đường bất đồng bộ trả 202 sau khi SQS đã nhận body. Việc hai mã HTTP đều ở nhóm 2xx chỉ cho biết request đã đạt điều kiện thành công của endpoint tương ứng.

Nếu SES từ chối sau khi nhánh đồng bộ đã ghi `RECEIVED`, API có thể trả 500 trong khi bảng vẫn có item. Vì vậy “tìm thấy bản ghi” và “đơn hoàn tất” phải có cột riêng trong báo cáo. Với nhánh bất đồng bộ, một lỗi ở workflow có thể xảy ra sau khi máy khách đã nhận 202. Đọc trạng thái cuối theo đúng tiền tố mới xác định được kết quả.

## Đặt cùng quy ước thời gian cho các hàm

Trên nhánh `/async`, `acceptedAt` được ghi trước khi gửi SQS và đi trong body đến worker. Worker dùng nó làm `createdAt` nếu có. Tin gửi thẳng từ SQS ở các phần trước có thể thiếu trường này, nên vẫn giữ phương án dùng thời điểm hiện tại. Trên nhánh `/sync`, thời điểm bắt đầu nằm trong handler gọi trực tiếp. Vì thế độ trễ API đo tại máy khách và khoảng thời gian tính từ các trường trong bảng có hai điểm bắt đầu khác nhau.

Sau khi ghi ba tệp, kiểm tra cú pháp tại máy. Việc kiểm tra import và quyền gọi dịch vụ được xác nhận tiếp ở bước build và phép thử nhỏ sau deploy.

```powershell
node --check .\src\sync-order.mjs
node --check .\src\async-order.mjs
node --check .\src\index.mjs
```
