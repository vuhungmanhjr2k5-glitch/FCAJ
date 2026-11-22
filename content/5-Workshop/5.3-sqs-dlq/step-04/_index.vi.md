---
title: "Thử để tin nhắn rơi vào DLQ"
weight: 4
chapter: false
pre: " <b> 5.3.4 </b> "
---

[← Mục lục phần này](../)

## Mục đích và điều kiện thực hiện

Thử nghiệm mô phỏng consumer nhận thông điệp nhưng không xóa sau khi nhận. Khi số lượt nhận thực tế vượt ngưỡng của redrive policy, SQS có thể chuyển thông điệp sang DLQ để giữ lại phục vụ điều tra.

Thực hiện trên cặp queue của bài lab, trước khi gắn Lambda ở mục 5.4. Nếu consumer khác đang hoạt động, nó có thể nhận hoặc xóa thông điệp trước thao tác thủ công. Khi đó kết quả chịu ảnh hưởng của nhiều consumer cùng nhận, không còn là phép thử độc lập này.

Trước khi gửi, mở hàng đợi chính trong SQS tại Singapore, kiểm tra tab **Dead-letter queue** trỏ đúng OrdersDLQ và **Maximum receives = 3**. Ảnh cấu hình chỉ chứng minh redrive policy, chưa chứng minh thông điệp đã chuyển sang DLQ.

## 1. Lấy địa chỉ hai hàng đợi

Chạy các khối lệnh trong cùng một phiên PowerShell. Dùng Outputs của đúng stack để tránh kiểm tra nhầm hàng đợi.

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
[pscustomobject]@{ QueueUrl = $q; DlqUrl = $dlq } | Format-List
```

Kiểm tra số lượng thông điệp trong DLQ trước khi gửi để nhận biết dữ liệu tồn từ lần thử cũ:

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read DLQ attributes.' }
```

## 2. Gửi thông điệp có mã nhận diện riêng

Body được thêm hậu tố ngẫu nhiên để phân biệt các lần thử. Lưu cả Body và MessageId từ kết quả dưới đây để đối chiếu về sau.

```powershell
$testBody = "test-dlq-$([guid]::NewGuid().ToString('N').Substring(0,10))"
$sendResult = aws sqs send-message --queue-url $q --message-body $testBody --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot send test message.' }
$sent = $sendResult | Out-String | ConvertFrom-Json
[pscustomobject]@{ Body = $testBody; MessageId = $sent.MessageId } | Format-List
```

## 3. Nhận lại thông điệp mà không xóa

Tham số `--visibility-timeout 0` áp dụng cho lần nhận thử, giúp thông điệp nhanh chóng đủ điều kiện nhận lại; nó không đổi cấu hình Visibility timeout mặc định 60 giây của hàng chính. Không gọi DeleteMessage trong vòng lặp.

```powershell
1..4 | ForEach-Object {
    Write-Host "Receive attempt $_"
    aws sqs receive-message --queue-url $q --visibility-timeout 0 --wait-time-seconds 2 --message-system-attribute-names ApproximateReceiveCount --region ap-southeast-1 --output json --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw 'Cannot receive messages.' }
    Start-Sleep -Seconds 1
}
```

Bốn lần gọi lệnh không bảo đảm có bốn lần giao cùng thông điệp. Poll rỗng là kết quả có thể xảy ra với hàng Standard. Chỉ đối chiếu những lần trả về đúng Body hoặc MessageId của thông điệp thử, cùng thuộc tính ApproximateReceiveCount nếu có.

## 4. Kiểm tra DLQ và đối chiếu kết quả

Đợi khoảng một phút rồi chạy lệnh sau. Khoảng chờ này là gợi ý quan sát, không phải cam kết thông điệp luôn chuyển xong trong một phút.

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read DLQ attributes.' }
```

Để kiểm tra nội dung, vào **SQS → đúng hàng đợi DLQ → Send and receive messages → Poll for messages**, mở thông điệp và đối chiếu Body hoặc MessageId đã lưu. Khi chụp ảnh, giữ tên DLQ và nội dung thông điệp trong cùng khung hình.

> Số liệu SQS là xấp xỉ và có thể cập nhật trễ. Số lượng bằng 1 có thể là thông điệp tồn từ trước; một lần receive rỗng cũng chưa đủ chứng minh thông điệp vừa chuyển sang DLQ. Poll trên Console là một lần nhận và có thể làm thông điệp tạm thời không còn visible.

## Minh chứng từ lần thử trước

![Kết quả nhận lại thông điệp và truy vấn số lượng trong hàng đợi đích](../../../../images/5-Workshop/5.3-SQS-DLQ/dlq-receive-result.png)

*PowerShell ghi nhận cùng MessageId với Body `test-dlq` ở các lần nhận 2 và 3. Sau thời gian chờ, truy vấn hàng đợi trong biến `$dlq` trả về `ApproximateNumberOfMessages = 1`.*

Ảnh được ghi nhận ở phiên bản dùng Body cố định `test-dlq`. Hướng dẫn hiện tại dùng `test-dlq-<mã ngẫu nhiên>` để dễ nhận diện từng lần thử. Vì vậy, ảnh minh họa cách đọc kết quả của lần thử trước, không phải kết quả của mã ngẫu nhiên vừa được tạo.

Ảnh hỗ trợ minh chứng việc nhận lại thông điệp và kiểm tra số lượng hàng đích. Để xác nhận đúng thông điệp đã vào DLQ, cần đối chiếu thêm tên hàng đợi đích với Body hoặc MessageId. Không kết luận chỉ từ Receive count bằng 4 hoặc bộ đếm DLQ lớn hơn 0.

## Cách ghi nhận kết quả

- Nếu tìm thấy đúng thông điệp trong DLQ, ghi Body, MessageId, tên DLQ và thời điểm quan sát.
- Nếu hàng chính vẫn còn thông điệp, kiểm tra các lần nhận thực tế và redrive policy trước khi poll thêm.
- Nếu cả hai bộ đếm tạm thời bằng 0, chờ cập nhật và đối chiếu kết quả receive; chưa kết luận thông điệp bị mất.
- Nếu thiếu bằng chứng xác định hàng đích hoặc thông điệp, ghi kết quả chưa được xác nhận đầy đủ.

Thực nghiệm này kiểm tra hành vi nhận lại khi consumer nhận rồi để nguyên thông điệp. Ở phần Lambda tiếp theo, nguyên nhân nhận lại có thể là xử lý lỗi; đó là một kịch bản khác cần được đối chiếu riêng.
