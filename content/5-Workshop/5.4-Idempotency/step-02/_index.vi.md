---
title: "Viết mã Lambda"
weight: 2
chapter: false
pre: " <b> 5.4.2 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Tạo handler trong thư mục ứng dụng

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

```powershell
$content = @'
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async (event) => {
  for (const record of event.Records) {
    const order = JSON.parse(record.body); // JSON sai se gay loi -> retry -> DLQ

    try {
      await db.send(new PutCommand({
        TableName: process.env.TABLE_NAME,
        Item: {
          orderId: order.orderId,
          status: "PROCESSING",
          createdAt: Date.now(),
        },
        // Chi ghi neu orderId chua ton tai
        ConditionExpression: "attribute_not_exists(orderId)",
      }));
    } catch (e) {
      if (e.name === "ConditionalCheckFailedException") {
        console.log("Don trung, bo qua:", order.orderId);
        continue;
      }
      throw e;
    }

    console.log("Da xu ly:", order.orderId);
  }
};
'@
$path = Join-Path $PWD.Path 'src/index.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

Điểm mấu chốt là `ConditionExpression`: DynamoDB chỉ cho ghi nếu `orderId`
chưa tồn tại, nên lần ghi thứ hai sẽ bị từ chối và Lambda bỏ qua đơn đó.

> Ở mục 5.5 ta sẽ bổ sung trạng thái đơn (`PROCESSING` → `DONE`) để xử lý
> trường hợp đã ghi sổ nhưng bước gửi email thất bại.

### Đóng gói thư viện cùng mã Lambda

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

## Vì sao điều kiện nằm ngay trên lệnh ghi

Giả sử hai invocation cùng nhận `A001`. Nếu mỗi invocation đọc bảng trước rồi mới quyết định ghi, cả hai có thể đọc lúc item chưa tồn tại. Ở đây DynamoDB đánh giá `attribute_not_exists(orderId)` ngay khi thực hiện Put. Một lần ghi được chấp nhận, lần còn lại nhận `ConditionalCheckFailedException`. Vì phép kiểm tra gắn với đúng item, hai đơn có mã khác nhau vẫn xử lý độc lập.

Đọc khối catch theo từng nhánh. Khi gặp lỗi điều kiện, worker ghi log với mã đơn và tiếp tục đến cuối batch. Tích hợp SQS xem lần xử lý đó là hoàn tất. Với lỗi khác, worker ném lỗi ra ngoài để tin có thể được giao lại. Nếu nuốt cả lỗi quyền DynamoDB như một trường hợp trùng, một đơn mới có thể rời hàng chính dù chưa được ghi nhận.

## Xác định giới hạn của phiên bản này

Sau một lần Put thành công, dữ liệu có `createdAt` và `PROCESSING`. Giả sử tiến trình dừng ngay sau lần ghi đó. Lần giao kế tiếp sẽ thấy khóa đã tồn tại. Ở phiên bản này chưa có workflow để tiếp tục gửi thư từ chỗ dừng, nên chỉ có thể kết luận rằng bài đã tạo được một dấu ghi nhận. Mục 5.5 bổ sung trạng thái `RECEIVED` và tên execution để quản lý bước khởi động sau khi ghi.

Có thể kiểm tra cú pháp JavaScript tại máy sau khi ghi file. Phép kiểm tra này chưa gọi handler và chưa đọc SQS.

```powershell
node --check .\src\index.mjs
```
