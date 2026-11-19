---
title: "Worklog Tuần 6"
date: 2026-09-14
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

**Thời gian thực tập:** 14/09/2026 - 19/09/2026

### Mục tiêu tuần 6:

- Sử dụng hàng đợi để tách tốc độ tiếp nhận yêu cầu khỏi tốc độ xử lý.
- Hiểu cơ chế gửi lại thông điệp và cách cô lập thông điệp xử lý thất bại.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 14/09/2026 | Tìm hiểu SQS Standard, cơ chế giao thông điệp ít nhất một lần và khả năng thông điệp đến không đúng thứ tự. | [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) |
| 15/09/2026 | Tạo hàng đợi xử lý đơn hàng và cấu hình quyền để EventBridge gửi thông điệp vào hàng đợi. | [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) |
| 16/09/2026 | Kết nối SQS với Lambda consumer; rà soát batch size, visibility timeout và giới hạn xử lý đồng thời. | [Using Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html) |
| 18/09/2026 | Cấu hình dead-letter queue và ngưỡng số lần nhận; mô phỏng lỗi để theo dõi thông điệp được thử lại và chuyển sang hàng đợi lỗi. | [SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) |
| 19/09/2026 | Thực hành báo cáo lỗi từng thông điệp trong batch và ghi lại cách kiểm tra trước khi redrive thông điệp lỗi. | [SQS error handling in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html) |

### Kết quả đạt được tuần 6:

- Xây dựng luồng thử nghiệm nhận thông điệp từ SQS và xử lý bằng Lambda.
- Hiểu vai trò của retry và dead-letter queue, cùng yêu cầu chống tác động lặp khi thông điệp được giao lại.