---
title: "Thử để tin nhắn rơi vào DLQ"
weight: 4
chapter: false
pre: " <b> 5.3.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Lấy địa chỉ hai hàng đợi vào biến:

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
```

Gửi một tin nhắn thử:

```powershell
$testBody = "test-dlq-$([guid]::NewGuid().ToString('N').Substring(0,10))"
$testBody
aws sqs send-message --queue-url $q --message-body $testBody --region ap-southeast-1 --output json --no-cli-pager
```

Nhận tin nhắn nhiều lần mà **không xóa** (tham số `--visibility-timeout 0` làm tin nhắn hiện lại ngay):

```powershell
1..4 | ForEach-Object {
  aws sqs receive-message --queue-url $q --visibility-timeout 0 --wait-time-seconds 2 --message-system-attribute-names ApproximateReceiveCount --region ap-southeast-1 --output json --no-cli-pager
  Start-Sleep -Seconds 1
}
```

Khi số lượt nhận thực tế vượt ngưỡng, SQS có thể chuyển tin sang DLQ. Đợi khoảng một phút rồi kiểm tra DLQ:

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages --region ap-southeast-1 --output json --no-cli-pager
```

Hoặc vào Console → hàng đợi DLQ → **Send and receive messages** → **Poll for messages**.

![Tin nhắn trong DLQ](/images/5-Workshop/5.3-SQS-DLQ/dlq-message.png)

> Các số liệu của SQS là xấp xỉ và có thể trễ vài chục giây.

## Chuẩn bị một phép thử có thể nhận diện

Thực hiện trước khi gắn Lambda ở mục 5.4, trên cặp queue dành cho bài lab. Nếu đã có consumer khác, consumer có thể xóa tin ngay sau khi bạn gửi. Khi đó kết quả quan sát thuộc một cuộc tranh nhận giữa hai consumer. Dùng một body có mã riêng cho lần thử và ghi lại `MessageId` mà SQS trả về. Một tin cũ trong DLQ có thể làm bộ đếm dương ngay cả trước lần gửi mới.

Khối nhận lặp dùng thời gian ẩn bằng 0 cho từng lần nhận thử. Điều này giúp tin vừa nhận nhanh chóng đủ điều kiện nhận lại và giữ nguyên cấu hình 60 giây của hàng chính. Không xóa tin trong vòng lặp. Mỗi lần nhận được body tương ứng mới được tính là một lần giao. Poll rỗng cũng là một kết quả hợp lệ của hàng Standard.

## Đọc kết quả khi vòng lặp kết thúc

Nếu thấy body mới ở DLQ, ghi mã chạy, `MessageId` và tên hàng đích vào một dòng. Nếu hàng chính vẫn còn tin sau vòng đầu, kiểm tra nhận được bao nhiêu lần thực sự và kiểm tra redrive policy trước khi poll thêm. Nếu cả hai hàng tạm thời đều báo 0, chờ bộ đếm cập nhật. Có thể dùng Console xem body trong DLQ, nhưng mỗi lần poll cũng là một lần nhận và có thể làm tin tạm biến mất khỏi danh sách visible.

Một lần chuyển DLQ chỉ chứng minh hành vi nhận lại của thông điệp đã thử. Ở chương kế tiếp, nguyên nhân nhận lại sẽ là Lambda xử lý lỗi. Trong bài hiện tại, chính thao tác nhận rồi để nguyên tin đã tạo ra điều kiện đó.
