---
title: "Điều phối và thông báo"
weight: 5
chapter: false
pre: " <b> 5.5 </b> "
---

<!-- workshop-split:v1 -->

## Mục tiêu

Thay vì để một hàm Lambda làm hết mọi việc, ta dùng **AWS Step Functions** để
điều phối quy trình xử lý đơn hàng. Tác vụ gửi email có cơ chế **thử lại (Retry)** và
**bắt lỗi (Catch)**, trạng thái từng đơn được ghi lại, và quản trị viên
được báo ngay qua **Amazon SNS** khi có đơn thất bại.

Quy trình:

1. Lambda nhận tin nhắn từ SQS, ghi sổ DynamoDB (`RECEIVED`) và khởi động state machine.
2. State machine gửi email xác nhận cho khách bằng **Amazon SES** (thử lại tối đa 3 lần).
3. Thành công: cập nhật trạng thái `DONE`.
4. Thất bại: cập nhật `FAILED`, gửi cảnh báo qua SNS, kết thúc ở trạng thái lỗi.

![Sơ đồ quy trình](/images/5-Workshop/5.5-Orchestration/workflow.png)

## Các phần thực hành

Mở từng mục dưới đây hoặc dùng menu bên trái. Thực hiện theo thứ tự; các bước sau có thể dùng tài nguyên và cấu hình từ bước trước.

- [5.5.1 Thay toàn bộ template.yaml](step-01/)
- [5.5.2 Thay toàn bộ src/index.mjs](step-02/)
- [5.5.3 Triển khai](step-03/)
- [5.5.4 Kiểm thử](step-04/)
- [5.5.5 Kiểm tra kết quả](step-05/)
- [5.5.6 Xử lý sự cố](step-06/)
