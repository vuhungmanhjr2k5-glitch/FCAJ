---
title: "Mục tiêu và kiến trúc giám sát"
weight: 1
chapter: false
pre: " <b> 5.6.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Theo dõi SQS, Lambda và Step Functions bằng CloudWatch metric, ba alarm, một dashboard và email SNS tại **ap-southeast-1**. Stack giám sát bổ sung là **order-monitoring-evidence**, dùng tài nguyên ứng dụng của **order-orchestration-evidence**.

## Theo dõi một đơn và theo dõi cả hệ thống

Để tìm một đơn cụ thể, dùng khóa DynamoDB và lịch sử execution. CloudWatch trả lời một câu hỏi rộng hơn: trong một khoảng thời gian, thành phần nào đang có dấu hiệu bất thường? Khi Lambda có một invocation lỗi, metric Errors có thể tăng nhưng datapoint không chứa orderId. Hãy giữ thời gian, tên hàm và Region để quay lại log khi cần truy tìm thông điệp.

Dashboard theo dõi bốn nhóm tín hiệu. Số tin visible cho biết còn bao nhiêu tin có thể được nhận; tuổi của tin cũ nhất giúp phát hiện tình trạng chờ lâu; nhóm execution cho thấy workflow bắt đầu và kết thúc thế nào; nhóm Lambda cho thấy số invocation và số lỗi. Khi đọc cùng nhau, backlog tăng cùng tuổi tăng gợi ý nguồn tin đến nhanh hơn tốc độ xử lý trong khoảng đang xem.

## Lắp phần giám sát vào stack của bạn

Phần triển khai lại trong bài dùng stack ứng dụng `order-system` và một stack giám sát riêng `order-monitoring`. Stack giám sát nhận tên queue, tên hàm, ARN state machine và ARN topic từ stack ứng dụng. Các tên có hậu tố `evidence` trên hình thuộc lần đo đã ghi trong tài liệu. Khi chạy trên tài khoản của bạn, lấy giá trị từ Resources và Outputs để theo dõi đúng nơi có lưu lượng.

Alarm có thể gửi thông báo khi đổi trạng thái trong khi workflow có thể gửi SNS cho từng đơn thất bại. Trong hộp thư, một thông báo CloudWatch nêu tên alarm và ngưỡng, còn thông báo nghiệp vụ nêu mã đơn. Ghi rõ nguồn trước khi dùng bất kỳ email nào để đo thời gian.

### Một ví dụ về lựa chọn biểu đồ đầu tiên

Khách báo chờ lâu nhưng hoàn toàn chưa có cảnh báo lỗi. Hãy xem tuổi tin cũ nhất và số tin hàng chính. Nếu cả hai tăng trong khi execution thành công vẫn xuất hiện, worker có thể đang hoàn tất công việc chậm hơn tốc độ tiếp nhận. Nếu hàng chính ít tin nhưng có nhiều execution đang chờ retry, điểm chờ nằm sau hàng đợi. Ba alarm hiện tại nhắm vào lỗi cụ thể, nên một tình huống xử lý chậm vẫn cần đọc dashboard.
