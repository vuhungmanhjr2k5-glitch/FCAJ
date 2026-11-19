---
title: "Kiểm thử"
weight: 4
chapter: false
pre: " <b> 5.4.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Chuẩn bị mã và hai đích kiểm tra

```powershell
$stackData = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read order-system.' }
$outputs = @{}
foreach ($item in ($stackData | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$q = $outputs['QueueUrl']
if (-not $q) { throw 'Missing output: QueueUrl' }
$dlq = $outputs['DlqUrl']
if (-not $dlq) { throw 'Missing output: DlqUrl' }
$table = $outputs['TableName']
if (-not $table) { throw 'Missing output: TableName' }
```

### 1. Gửi đơn và đọc item theo khóa

```powershell
$orderId = 'A-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$file = Join-Path $PWD.Path ("{0}.json" -f $orderId)
$body = @{ orderId = $orderId } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
```

Đợi worker có thời gian xử lý. Nếu lần đọc đầu chưa có `Item`, chạy lại riêng khối đọc dưới đây.

```powershell
$keyFile = Join-Path $PWD.Path 'workshop-item-key.json'
$keyJson = @{ orderId = @{ S = $orderId } } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($keyFile, $keyJson, [Text.UTF8Encoding]::new($false))
aws dynamodb get-item --table-name $table --key "file://$keyFile" --consistent-read --region ap-southeast-1 --output json --no-cli-pager
```

### 2. Gửi lại chính tệp vừa tạo

```powershell
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
sam logs --stack-name order-system --name ProcessOrderFn --region ap-southeast-1 --tail
```

Chờ log báo trùng mang mã `$orderId`. Nhấn **Ctrl+C** để kết thúc theo dõi log.

### 3. Theo dõi một body không đọc được

```powershell
$badBody = "not-json-$orderId"
aws sqs send-message --queue-url $q --message-body $badBody --region ap-southeast-1 --output json --no-cli-pager
```

Đợi qua các lần giao lại, thường vài phút với cấu hình của bài. Mở DLQ chỉ ra bởi `$dlq` và tìm đúng body. Ở lượt ghi hình cũ của stack `order-idempotency-evidence`, body là `khong-phai-json` và mã đơn hợp lệ là `A001`.

## Đặt thứ tự để phép thử trùng có ý nghĩa

Lần gửi đầu tiên tạo item. Chờ đến khi đọc được `PROCESSING` và ghi lại `createdAt` trước khi gửi lần thứ hai. Dùng cùng tệp JSON cho lần gửi lại. Nếu thay mã giữa hai lần, bạn đang kiểm tra hai đơn hợp lệ. Ngược lại, nếu tái sử dụng `A001` sau buổi thực hành trước, ngay cả lần đầu của buổi mới cũng có thể rơi vào nhánh trùng. Hãy dùng mã mới cho mỗi lượt độc lập.

Một dòng trong DynamoDB sau hai lần gửi có thể xuất hiện cả khi ứng dụng vô tình ghi đè cùng khóa. Để kiểm tra nhánh chống trùng, kết hợp log có đúng mã đơn với thời điểm tạo giữ nguyên. Với bài này, một conditional write thành công và một lỗi điều kiện được xử lý sẽ tạo đúng cặp tín hiệu đó.

## Phân biệt lỗi cú pháp với lỗi ghi bảng

Body dạng văn bản ở trường hợp 3 sẽ lỗi ngay tại JSON.parse, trước khi Lambda có thể lấy `orderId`. Vì thế hãy dùng chuỗi nhận diện trong body và thời điểm gửi để tìm nó trong DLQ. Một exception truy cập DynamoDB cũng có thể khiến Lambda thử lại, nhưng log sẽ nêu action hoặc tên lỗi khác. Đọc dòng lỗi gốc trước khi sửa cấu hình bảng. Khi đã thu được body hoặc log phù hợp, ghi kết quả bằng văn bản bên cạnh mã lượt thử.

### Đọc một item từ AWS CLI

Trong kết quả GetItem, DynamoDB dùng ký hiệu kiểu dữ liệu. `orderId.S` chứa mã dạng chuỗi, `status.S` mang giá trị `PROCESSING` và `createdAt.N` chứa số mili giây dạng chuỗi của API DynamoDB. Hai lượt đọc có cùng khóa và cùng giá trị `createdAt.N` là một dấu hiệu item đầu được giữ lại. Kết hợp dấu hiệu đó với dòng log báo trùng sẽ giải thích được quyết định của Lambda ở lần gửi thứ hai.

### Hình trong lần thực nghiệm gốc

- [DynamoDB ghi nhận đơn A001 với trạng thái PROCESSING](/images/5-Workshop/5.4-Idempotency/dynamodbb-item.png)
- [Lambda skips the resubmitted order A001](/images/5-Workshop/5.4-Idempotency/duplicate-logg.png)
- [DynamoDB retains one A001 record after resubmission](/images/5-Workshop/5.4-Idempotency/duplicate-item.png)
- [Thông điệp không hợp lệ xuất hiện trong DLQ](/images/5-Workshop/5.4-Idempotency/dlqq-message.png)
