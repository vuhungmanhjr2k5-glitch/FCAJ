---
title: "Kiểm tra kết quả"
weight: 5
chapter: false
pre: " <b> 5.4.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

| Trường hợp | Kết quả mong đợi |
|---|---|
| Đơn hợp lệ | Có một dòng trong DynamoDB |
| Gửi trùng | Log báo trùng, không thêm dòng mới |
| Tin nhắn không phải JSON | Sau vài lần thử, tin nhắn nằm trong DLQ |

## Lập một dòng cho mỗi tình huống

Điền bảng bằng giá trị từ lượt chạy của bạn. `MessageId` là dấu vết của lần gửi, còn `orderId` là khóa nghiệp vụ. Ở tình huống gửi lại sẽ có hai MessageId khác nhau dù chỉ có một orderId. Với tin hỏng cú pháp, để trống cột mã đơn và ghi chuỗi body thay vào đó. Cách ghi này vẫn cho phép lần theo kết quả khi Lambda không tạo được item.

| Dữ kiện cần lưu | Đơn mới | Gửi lại cùng đơn | Body sai |
|---|---|---|---|
| Dấu nhận diện | orderId mới, MessageId đầu | Cùng orderId, MessageId thứ hai | Chuỗi body, MessageId |
| Thay đổi trong bảng | Có `createdAt` và `PROCESSING` | So sánh với `createdAt` đầu | Parse dừng trước thao tác ghi |
| Dấu vết quyết định | Log xử lý đúng mã | Log đi vào nhánh trùng | Log parse và body ở đúng DLQ |

## Xử lý trường hợp kết quả còn thiếu

Nếu item đã xuất hiện nhưng chưa thấy log báo trùng, kiểm tra khoảng thời gian log, stream và tên hàm. Nếu log đã có mà kết quả đọc bảng còn cũ, thực hiện một lần đọc nhất quán theo khóa. Nếu thấy một tin trong DLQ nhưng body thuộc lượt trước, đánh dấu tình huống hiện tại đang chờ và theo dõi tiếp. Chỉ chuyển sang điều phối khi giải thích được ba đường đi. Phiên bản sau thay đổi ý nghĩa của các trạng thái, nên hãy ghi lại kết quả ở giai đoạn `PROCESSING` trước khi cập nhật stack.

### Đọc lại một đơn từ phiên PowerShell mới

Nhập mã vừa ghi trong bảng kết quả. Phép đọc dùng khóa chính và chế độ nhất quán mạnh. Nếu output không có `Item`, kiểm tra lại mã, bảng và Region trước khi gửi thêm tin.

```powershell
$orderId = Read-Host 'Order ID'
$table = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue | [0]" --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $table -eq 'None') { throw 'TableName unavailable.' }
$file = Join-Path $PWD.Path 'workshop-item-key.json'
$keyJson = @{ orderId = @{ S = $orderId } } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $keyJson, [Text.UTF8Encoding]::new($false))
aws dynamodb get-item --table-name $table --key "file://$file" --consistent-read --region ap-southeast-1 --output json --no-cli-pager
```
