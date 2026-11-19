---
title: "Thí nghiệm 2: gửi trùng đơn"
weight: 6
chapter: false
pre: " <b> 5.7.6 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Đơn **dup-proof-20261113-220937-a7468d** được gửi cùng nội dung **5 lần** vào SQS. Kết quả: execution **SUCCEEDED**, **1** tác vụ SES thành công trong execution, DynamoDB **DONE**, **emailCount = 1**.

![Kết quả kiểm tra gửi trùng](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-validation.png)

Log ghi nhận bỏ qua đơn trùng. Hai dòng báo khởi động không chứng minh có hai execution: Standard StartExecution cùng tên và input khi execution đang chạy có thể trả về execution hiện có.

[Log bỏ qua đơn trùng](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-skip-logs.png)

Hộp thư nhận email xác nhận đúng mã đơn. Kết quả hỗ trợ chống trùng trong lần thử này; không bảo đảm email được gửi đúng một lần trong mọi tình huống lỗi.

[Email xác nhận thực nhận](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-confirmation-email.png)

## Thử lại một đơn và giữ nguyên danh tính

Phép thử gốc dùng **một** body gửi năm lần trực tiếp vào SQS. Điều này kiểm tra worker và workflow sau queue. API tiếp nhận không tham gia phép thử đó. Để làm lại, chọn hộp thư thử đã xác minh nếu muốn kiểm tra nhận thư, hoặc dùng simulator nếu chỉ cần xem kết quả tác vụ SES. Giữ nguyên cả `orderId` và nội dung giữa năm lần gửi. Một giá trị ngẫu nhiên mới ở mỗi lần sẽ làm thay đổi câu hỏi thực nghiệm.

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue | [0]" --region ap-southeast-1 --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $q -eq 'None') { throw 'QueueUrl unavailable.' }
$dupId = 'dup-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$email = Read-Host 'Verified test recipient (Enter = SES simulator)'
if ([string]::IsNullOrWhiteSpace($email)) { $email = 'success@simulator.amazonses.com' }
$body = @{ orderId = $dupId; customerEmail = $email } | ConvertTo-Json -Compress
$file = Join-Path $PWD.Path "$dupId.json"
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
1..5 | ForEach-Object {
    aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --query MessageId --output text --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw "Send $_ failed." }
}
$dupId
```

Mỗi lần SQS chấp nhận sẽ trả một MessageId. Đó là năm lần gửi độc lập cùng mô tả một ý định mua hàng. Tìm execution theo mã `$dupId` rồi đếm tác vụ gửi SES thành công trong history của execution đó. Tiếp theo đọc item cùng mã, ghi `status` và `emailCount`. Nếu dùng hộp thư thật, tìm đúng tiêu đề của mã vừa tạo. Không cần gửi thêm lượt thứ sáu chỉ để kiểm tra lại kết quả lượt này.

## Nếu số đếm không khớp

Có hai dòng “khởi động” trong log thì trước hết đếm **execution ARN khác nhau**. Có một execution nhưng hai tác vụ SES thành công thì đọc state history, thời điểm và đường retry. Có một item nhưng bộ đếm lớn hơn 1 thì kiểm tra nó thuộc đúng phiên bản template và đúng mã lượt hay chưa. Những quan sát này cho biết điểm nào cần điều tra tiếp. Chỉ báo số thư thực nhận nếu đã kiểm tra hộp thư tương ứng.
