---
title: "Kiểm tra trên Console"
weight: 3
chapter: false
pre: " <b> 5.3.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

1. Vào **Amazon SQS** → **Queues**.
2. Bạn thấy hai hàng đợi có tên bắt đầu bằng `order-system-`.
3. Mở hàng đợi chính → tab **Dead-letter queue**: hàng đợi lỗi đã được gắn, **Maximum receives = 3**.

![Hàng đợi SQS](/images/5-Workshop/5.3-SQS-DLQ/queues.png)

## So sánh giá trị đang chạy với mẫu đã triển khai

Mở hàng đợi chính bằng tên vật lý trong `QueueUrl`. Trên tab cấu hình, lấy ARN của DLQ mà redrive policy trỏ tới. Tiếp theo mở queue trong `DlqUrl` và đọc ARN của chính hàng đó. Hai ARN phải giống nhau từng ký tự. Đuôi tên trông giống nhau chưa đủ, vì tài khoản có thể giữ hàng thử từ buổi trước.

```powershell
$raw = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the application stack.' }
$outputs = ($raw | Out-String | ConvertFrom-Json).Stacks[0].Outputs
$q = ($outputs | Where-Object OutputKey -eq 'QueueUrl').OutputValue
$dlq = ($outputs | Where-Object OutputKey -eq 'DlqUrl').OutputValue
$main = aws sqs get-queue-attributes --queue-url $q --attribute-names RedrivePolicy VisibilityTimeout --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the source queue.' }
$attrs = ($main | Out-String | ConvertFrom-Json).Attributes
$policy = $attrs.RedrivePolicy | ConvertFrom-Json
$actualArn = aws sqs get-queue-attributes --queue-url $dlq --attribute-names QueueArn --query 'Attributes.QueueArn' --region ap-southeast-1 --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the DLQ.' }
[pscustomobject]@{
    ConfiguredTarget  = $policy.deadLetterTargetArn
    ActualTarget      = $actualArn
    SameTarget        = ($policy.deadLetterTargetArn -eq $actualArn)
    MaxReceives       = $policy.maxReceiveCount
    VisibilitySeconds = $attrs.VisibilityTimeout
}
```

Cột `SameTarget` phải là `True`. `MaxReceives` được trả về từ JSON cấu hình và cần có giá trị 3, còn `VisibilitySeconds` là 60. Trong AWS CLI, thuộc tính `RedrivePolicy` nằm dưới dạng chuỗi JSON bên trong kết quả JSON bên ngoài. Hai lần đọc JSON trong khối lệnh tương ứng với hai lớp này.

## Kiểm tra theo triệu chứng

Nếu có đủ hai URL mà `RedrivePolicy` trống, kiểm tra xem đã chọn nhầm DLQ làm hàng chính hay chưa. Nếu ARN trỏ đến một hàng khác, mở tab **Resources** của stack để xác định logical ID tương ứng với từng tên vật lý. Nếu lệnh báo queue không tồn tại, đọc lại stack Outputs tại Singapore và kiểm tra profile đang dùng. Chỉ sang phép thử giao lại khi đã xác định đúng đích. Một dòng kết quả ghi hai ARN và ngưỡng nhận đủ mô tả cấu hình cho lần thực hành này.

### Nếu phải bàn giao cặp queue cho người khác

Hãy cung cấp tên stack, Region và tên hai output. Người nhận bàn giao có thể tự đọc URL hiện hành bằng profile của họ, rồi chạy truy vấn thuộc tính trên cả hai queue. Khi stack được tạo lại, URL cũ lưu trong tệp cá nhân có thể không còn dùng được. Tham chiếu qua Outputs giúp nhận ra điều này trước lần gửi đầu tiên. Nếu `SameTarget` bằng `False`, khoanh vùng ở bước đấu nối hai queue trước khi thử lỗi.
