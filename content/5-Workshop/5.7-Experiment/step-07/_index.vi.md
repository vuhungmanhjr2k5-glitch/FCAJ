---
title: "Thí nghiệm 3: lỗi và cảnh báo"
weight: 7
chapter: false
pre: " <b> 5.7.7 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

### Lỗi SES và thử lại

Đơn **failure-proof-20261113-222244-4343a2** dùng địa chỉ **not-an-email**. SES trả **SesV2.BadRequestException**, HTTP **400**, nguyên nhân **Missing final '@domain'**. Event history cho thấy tác vụ được lập lịch và thực thi lại sau lỗi. SDK Attempt Count: 1 không phải tổng số lần Step Functions thử lại.

![Nguyên nhân lỗi SES và lịch sử thử lại](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-ses-error-history.png)

### Nhánh xử lý lỗi

Catch chuyển sang **MarkFailed**, rồi **NotifyAdminFailure**; cả hai thành công. Quy trình kết thúc tại **OrderFailed**; MarkDone không chạy trên nhánh này.

[Nhánh xử lý lỗi của quy trình](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-workflow-graph.png)

Email SNS chứa đúng mã đơn, do bước NotifyAdminFailure gửi.

[Email cảnh báo từ quy trình](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/failure-admin-email.png)

### DLQ và CloudWatch Alarm

Phép thử riêng dùng thông điệp **not-json-dlq-proof-20261113-223327-eed369**, không phải JSON hợp lệ. Kiểm tra thông điệp trong DLQ **order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU**; ảnh cần hiển thị cả tên hàng đợi và nội dung.

[Thông điệp lỗi trong đúng DLQ](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-malformed-message.png)

Alarm **order-orchestration-evidence-dlq-proof** giám sát ApproximateNumberOfMessagesVisible của DLQ và chuyển sang **ALARM** lúc **13/11/2026 15:39:57 UTC**.

[Alarm DLQ được kích hoạt](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-cloudwatch-alarm.png)

Email CloudWatch qua SNS ghi cùng tên alarm và mốc chuyển trạng thái, datapoint **1** vượt ngưỡng **0**. Đây là cảnh báo CloudWatch, khác email do workflow gửi ở trên. Bộ ảnh này chưa đo độ trễ cảnh báo.

[Email cảnh báo CloudWatch qua SNS](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/dlq-alarm-email.png)

## Hai lỗi, hai đường báo khác nhau

Địa chỉ `not-an-email` vẫn là giá trị chuỗi trong một JSON hợp lệ. Lambda có thể ghi đơn và khởi động execution. Sau đó SES kiểm tra địa chỉ rồi trả lỗi. Ngược lại, body `not-json-...` làm parser của Lambda dừng ngay. Tin này không đi tới state gửi thư. Giữ riêng mã của hai trường hợp để không tìm nhầm một execution chưa từng được tạo.

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue | [0]" --region ap-southeast-1 --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $q -eq 'None') { throw 'QueueUrl unavailable.' }
$tag = [guid]::NewGuid().ToString('N').Substring(0,10)
$failedId = "ses-fail-$tag"
$badBody = "not-json-$tag"
$file = Join-Path $PWD.Path "$failedId.json"
$body = @{ orderId = $failedId; customerEmail = 'not-an-email' } | ConvertTo-Json -Compress
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --query MessageId --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'First send failed.' }
aws sqs send-message --queue-url $q --message-body $badBody --region ap-southeast-1 --query MessageId --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Second send failed.' }
[pscustomobject]@{ OrderId = $failedId; DlqMarker = $badBody }
```

Ở trường hợp thứ nhất, đọc event lỗi đầu, số lần task được lập lịch lại, trạng thái cập nhật bảng và kết quả publish của `NotifyAdminFailure`. MaxAttempts bằng 6 ở template 5.7 nên chừa thời gian cho backoff trước khi đọc trạng thái cuối. Nếu môi trường minh họa có bộ retry khác, đếm số sự kiện của đúng lượt đang xem. Ở trường hợp thứ hai, tìm chuỗi `$badBody` trong DLQ của cùng stack và kiểm tra tên alarm DLQ đã cấu hình ở 5.6.

## Ghi mốc thời gian có thể kiểm tra lại

Cần ba mốc nếu muốn tách phát hiện lỗi và chuyển thư: lỗi đầu trong history/log, lúc alarm chuyển trạng thái và lúc thư tới hộp thư. Chuyển hết về UTC trước khi trừ hai thời điểm. Trường `Timestamp` của SNS mô tả thời điểm thông báo của dịch vụ và có thể khác thời gian thư được nhận. Nếu DLQ đã ALARM từ trước, tin mới có thể nằm đúng DLQ mà không có email chuyển trạng thái mới. Ghi lại điều này như một kết quả hợp lệ của cơ chế cảnh báo.
