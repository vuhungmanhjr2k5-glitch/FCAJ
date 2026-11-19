---
title: "Kiểm thử"
weight: 4
chapter: false
pre: " <b> 5.5.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Chuẩn bị một lượt kiểm thử

```powershell
$stackData = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read order-system.' }
$outputs = @{}
foreach ($item in ($stackData | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$q = $outputs['QueueUrl']
if (-not $q) { throw 'Missing output: QueueUrl' }
$table = $outputs['TableName']
if (-not $table) { throw 'Missing output: TableName' }
$tag = [guid]::NewGuid().ToString('N').Substring(0,10)
$goodId = "B001-$tag"
$failId = "B002-$tag"
$email = Read-Host 'Verified test inbox for the order confirmation'
```

### 1. Theo dõi đơn thành công

```powershell
$file = Join-Path $PWD.Path ("{0}.json" -f $goodId)
$body = @{ orderId = $goodId; customerEmail = $email } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
$goodFile = $file
$goodId
```

Tìm tên `$goodId` trong Step Functions rồi đọc item. Khi execution thành công, kiểm tra `DONE` và tiêu đề email chứa mã này.

```powershell
$keyFile = Join-Path $PWD.Path 'workshop-item-key.json'
$keyJson = @{ orderId = @{ S = $goodId } } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($keyFile, $keyJson, [Text.UTF8Encoding]::new($false))
aws dynamodb get-item --table-name $table --key "file://$keyFile" --consistent-read --region ap-southeast-1 --output json --no-cli-pager
```

### 2. Thử lại cùng body

```powershell
aws sqs send-message --queue-url $q --message-body "file://$goodFile" --region ap-southeast-1 --output json --no-cli-pager
sam logs --stack-name order-system --name ProcessOrderFn --region ap-southeast-1 --tail
```

Đợi log của lần gửi trùng rồi nhấn **Ctrl+C**.

### 3. Theo nhánh SES từ chối

Trong SES sandbox, một địa chỉ người nhận chưa xác minh sẽ bị từ chối. Để có trường hợp lỗi cố định kể cả khi tài khoản đã ra khỏi sandbox, khối này dùng chuỗi sai định dạng.

```powershell
$email = 'not-an-email'
$file = Join-Path $PWD.Path ("{0}.json" -f $failId)
$body = @{ orderId = $failId; customerEmail = $email } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --output json --no-cli-pager
$failId
```

Tìm execution `$failId`. Sau các lần retry, kiểm tra `MarkFailed`, `NotifyAdminFailure`, item có `FAILED` và email quản trị mang mã đó. Bộ hình gốc minh họa đơn `B001` và `B002`.

## Theo dõi ba đường đi mà không lẫn mã đơn

Trước khi gửi, chọn một mã mới cho trường hợp thành công và một mã mới cho trường hợp SES từ chối. Các mã `B001`, `B002` trong ảnh thuộc lượt minh họa. Khối PowerShell dùng mã có hậu tố riêng để các lần thực hành không dùng lại tên execution cũ. Lưu body của lượt thành công và dùng đúng body đó khi thử trùng.

Khi thử đơn hợp lệ, phân biệt ba mốc: SQS trả MessageId, execution vào `Succeeded` và bảng ghi `DONE`. Bản mẫu 5.5 chỉ yêu cầu trường `status` ở bước cập nhật cuối. Một môi trường minh họa đã mở rộng có thể hiển thị thêm `doneAt` và `emailCount`. Hai trường này được đưa vào template của bài ở mục 5.7.1. Nếu muốn kiểm tra chuyển phát thực, dùng hộp thư thật đã xác minh. Simulator sẽ không cung cấp một hộp thư để mở.

## Đọc timeline của đơn bị từ chối

Trong **Execution event history**, mở sự kiện lỗi đầu ở tác vụ gửi SES, xem tên lỗi và thời điểm. Tìm lần lập lịch kế tiếp của cùng task rồi đi đến `MarkFailed`. Nếu trạng thái cuối là Failed, kiểm tra cả kết quả publish SNS trước đó. Bản ghi `FAILED` xác nhận được việc cập nhật vào bảng. ARN kết quả publish xác nhận yêu cầu tới topic. Email mang đúng mã đơn trong hộp thư mới giúp xác nhận người trực đã nhận thông báo.

Đừng dùng số tin đang nằm trong DLQ để kết luận về trường hợp này. Nếu Lambda đã khởi động được workflow, lỗi gửi thư xảy ra sau khi Lambda hoàn thành phần việc nhận tin từ SQS.

### Bảng ghi trong lúc chờ execution

| Trường cần điền | Đơn thành công | Đơn lỗi SES |
|---|---|---|
| Mã sẽ tra | Giá trị `$goodId` | Giá trị `$failId` |
| Sự kiện cần mở | Task SES thành công và cập nhật `MarkDone` | Lỗi đầu tại SES và lần chuyển sang `MarkFailed` |
| Dấu hiệu ở cuối history | `ExecutionSucceeded` | `ExecutionFailed` sau nhánh báo quản trị |
| Nguồn ngoài workflow | Item theo khóa, thư xác nhận trong inbox thử | Item theo khóa, thông báo SNS của đúng mã |

Nếu ở lần nhìn đầu chỉ thấy `ExecutionStarted`, hãy giữ nguyên cửa sổ và làm mới execution đó. Nếu đã thấy sự kiện cuối nhưng item chưa đúng, kiểm tra bạn đang đọc khóa của lượt mới, trong bảng cùng stack và cùng Region. Nếu có `MarkDone` mà khách chưa thấy thư, nhìn phản hồi từ SES và loại địa chỉ nhận trước khi thay đổi cấu hình gửi lại. Dấu thời gian trong history và trong hộp thư có thể khác múi giờ; hãy ghi kèm múi giờ khi bàn giao.

### Hình trong lần thực nghiệm gốc

- [Execution B001 có trạng thái Succeeded](/images/5-Workshop/5.5-Orchestration/b001-execution-succeeded.png)
- [Graph view thể hiện SendCustomerEmail và MarkDone thành công](/images/5-Workshop/5.5-Orchestration/b001-workflow-graph.png)
- [DynamoDB ghi nhận đơn B001 với trạng thái DONE](/images/5-Workshop/5.5-Orchestration/b001-dynamodb-done.png)
- [Execution thành công](/images/5-Workshop/5.5-Orchestration/execution-success.png)
- [Execution thất bại](/images/5-Workshop/5.5-Orchestration/execution-failed.png)
- [Email cảnh báo](/images/5-Workshop/5.5-Orchestration/alert-email.png)
