---
title: "Thay toàn bộ src/index.mjs"
weight: 2
chapter: false
pre: " <b> 5.5.2 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Tạo handler trong thư mục ứng dụng

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

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

> `orderId` dùng làm tên execution nên chỉ nên gồm chữ, số, dấu `-` và `_`
> (tối đa 80 ký tự).

### Đóng gói thư viện cùng mã Lambda

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

## Đọc lại một đơn đang ở RECEIVED

Lambda thực hiện hai thao tác riêng: ghi sổ đơn trong DynamoDB và yêu cầu Step Functions khởi động execution. Nếu tiến trình dừng ở khoảng giữa, item có thể còn ở `RECEIVED` khi SQS giao lại. Nhánh xử lý lỗi điều kiện đọc item cũ. Nếu trạng thái vẫn là `RECEIVED`, worker thử khởi động tiếp. Nếu item đã `DONE` hoặc `FAILED`, worker có thể kết thúc lần nhận lại. Trong đoạn mẫu, phép đọc theo khóa sau xung đột dùng chế độ nhất quán mạnh để thấy lần ghi vừa hoàn tất.

Tên execution được lấy từ `orderId`. Với Standard workflow đang chạy, gọi StartExecution lại với cùng tên và cùng input có thể trả lại kết quả của execution hiện có. Đổi input khi vẫn dùng tên cũ có thể sinh `ExecutionAlreadyExists`. Khi phát lại đúng một đơn, hãy giữ nguyên nội dung nghiệp vụ. Nếu khách cần sửa đơn đã xác nhận, đó là một yêu cầu nghiệp vụ riêng cần quy ước riêng.

## Ba tình huống để tự lần theo mã

| Lúc thông điệp đến | Đường đi qua handler | Điều cần tìm |
|---|---|---|
| Chưa có item | Put thành công, gọi StartExecution | Một execution mang tên mã đơn |
| Đã có item RECEIVED | Đọc lại trạng thái, thử khởi động với cùng tên | History của cùng execution |
| Item đã có trạng thái cuối | Bỏ qua lần giao lặp | Log gắn với mã đơn và item giữ trạng thái cuối |

Tên execution giúp kiểm soát việc khởi động trong phạm vi quy tắc đặt tên và thời gian lưu tên của Step Functions Standard. Bộ đếm email và trạng thái cuối vẫn do các bước phía sau ghi. Không thể suy ra số thư nhận được bằng cách đếm số dòng log “đã khởi động”.
