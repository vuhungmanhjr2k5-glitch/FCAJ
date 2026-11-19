---
title: "Xác minh email trong Amazon SES"
weight: 4
chapter: false
pre: " <b> 5.2.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Trong SES sandbox, dùng identity người gửi đã xác minh. Người nhận thử có thể là địa chỉ đã xác minh hoặc mailbox simulator của SES.

1. Vào *Amazon SES* → *Identities* → *Create identity*.
2. Chọn *Email address*, nhập email của bạn, chọn *Create identity*.
3. Mở email xác nhận từ AWS và bấm vào liên kết.
4. Trạng thái identity chuyển sang *Verified*.

![SES verified](/images/5-Workshop/5.2-Prerequisite/003.png)

<!-- workshop-detail:v1:begin -->
## Vì sao phải chuẩn bị email trước?

SES là dịch vụ gửi xác nhận cho người đặt hàng. Trạng thái Sandbox và việc xác minh identity được xét theo Region. Email đã xác minh ở một Region khác chưa đủ để sử dụng trong luồng đang chạy tại Singapore.

Khi SES còn Sandbox, người gửi và người nhận thông thường phải đáp ứng điều kiện xác minh. Các địa chỉ mailbox simulator của SES là ngoại lệ phục vụ kiểm thử. Khi được cấp Production access, người nhận không cần xác minh riêng, nhưng danh tính người gửi vẫn cần hợp lệ.

## Kiểm tra trạng thái tài khoản

```powershell
aws sesv2 get-account --region ap-southeast-1 --query '{Production:ProductionAccessEnabled,Sending:SendingEnabled}' --output json --no-cli-pager
```

Production=false cho biết tài khoản còn Sandbox. Production=true cho biết đã có quyền production; Sending cho biết trạng thái cho phép gửi của tài khoản. Lệnh này chỉ đọc thông tin, không gửi thư hoặc yêu cầu mở quyền.

## Chuẩn bị người nhận thử

Mở SES tại Singapore, chọn Verified identities và kiểm tra email. Nếu cần xác minh địa chỉ mới, tạo email identity rồi để chủ hộp thư mở liên kết AWS gửi. Chờ trạng thái Verified trước khi tạo đơn thử mới.

SNS subscription confirmation và SES identity verification là hai việc khác nhau: xác nhận nhận cảnh báo SNS không tự xác minh email trong SES. Địa chỉ success@simulator.amazonses.com giúp thử tác vụ gửi thành công nhưng không cung cấp một hộp thư Inbox để chụp email nhận.

<!-- workshop-detail:v1:end -->

<!-- workshop-proof:v1:begin -->
## Minh chứng xác minh email

{{< workshop-proof file="Verified.png" alt="Email identity có trạng thái Verified trong Amazon SES" caption="Amazon SES hiển thị email vuhungmanhjr2k5@gmail.com với loại Email address và trạng thái Verified." >}}

Ảnh xác nhận identity đã được xác minh trong danh sách đang xem. Phần ảnh này không hiển thị Region hoặc Production access. Cần kiểm tra riêng Region ap-southeast-1 và trạng thái Sandbox trước khi gửi đến địa chỉ khác. Verified không đồng nghĩa đã gửi hoặc nhận thành công một email xác nhận đơn hàng.
<!-- workshop-proof:v1:end -->

## Kiểm tra đúng địa chỉ sẽ dùng làm người gửi

Nhập địa chỉ email đã xác minh trong tài khoản thực hành. Lệnh dưới đọc trạng thái của email identity ở Singapore. Giá trị `VerifiedForSendingStatus` bằng `true` là điều cần thấy trước khi nhập địa chỉ này vào tham số `SenderEmail` ở bước triển khai. Ví dụ sử dụng một email identity riêng như các bước trên.

```powershell
$sender = Read-Host 'SES sender email'
aws sesv2 get-email-identity --email-identity $sender --region ap-southeast-1 --query '{Type:IdentityType,VerifiedForSendingStatus:VerifiedForSendingStatus}' --output table --no-cli-pager
```

Nếu kết quả là `NotFoundException`, kiểm tra lại Region và chính tả của địa chỉ. Nếu identity tồn tại nhưng chưa xác minh, mở hộp thư của địa chỉ vừa khai báo, tìm thư từ AWS và làm theo liên kết còn hiệu lực. Tránh tạo liên tiếp nhiều identity cho cùng một thao tác. Ghi lại email nào đóng vai trò gửi, email nào đóng vai trò nhận trong buổi thử.

## Chọn đúng kiểu kiểm thử email

Khi mục tiêu là đo thời gian API, địa chỉ simulator cho phép quan sát kết quả SES chấp nhận thư. Khi mục tiêu là chứng minh thư đến hộp thư, chọn hộp thư thử do bạn kiểm soát và tìm tiêu đề chứa mã đơn cụ thể. Khi mục tiêu là kiểm tra cảnh báo quản trị, hãy xem subscription SNS và danh tính của topic. Ba phép thử trả lời ba câu hỏi riêng ngay cả khi bạn dùng cùng địa chỉ cho hai vai trò.
