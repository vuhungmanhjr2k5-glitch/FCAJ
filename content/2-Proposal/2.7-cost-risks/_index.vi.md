---
title: "Chi phí, rủi ro và tài liệu tham khảo"
weight: 7
chapter: false
pre: " <b> 2.7. </b> "
---

## 1. Chi phí, rủi ro và biện pháp kiểm soát

### 1.1. Kiểm soát chi phí

Chi phí phụ thuộc vào Region, số yêu cầu, thời gian thực thi Lambda, lưu lượng SQS, thao tác DynamoDB và lượng log được lưu.

Trước khi kiểm thử tải, cần:

- Ước tính chi phí theo cấu hình và mức tải dự kiến.
- Thiết lập AWS Budgets và ngưỡng cảnh báo.
- Giới hạn tốc độ gửi yêu cầu và thời gian chạy.
- Đặt thời gian lưu log phù hợp.
- Theo dõi tài nguyên và dọn dẹp sau thử nghiệm.

Cảnh báo ngân sách không tự động dừng toàn bộ tài nguyên. Công cụ tạo tải cần có giới hạn và điều kiện dừng riêng.

Không giả định toàn bộ thử nghiệm sẽ miễn phí hoặc nằm trong Free Tier.

### 1.2. Rủi ro và biện pháp

| Rủi ro | Biện pháp dự kiến |
|--------|------------------|
| Thông điệp trùng hoặc đến không đúng thứ tự | Idempotency, kiểm soát chuyển trạng thái và kiểm thử giao lại. |
| Ghi dữ liệu thành công nhưng phát thông điệp thất bại | Lựa chọn cơ chế tiếp nhận tin cậy; xem xét outbox và đối soát. |
| Tồn đọng tăng nhanh | Theo dõi tuổi thông điệp, giới hạn tải và điều chỉnh năng lực consumer trong ngân sách. |
| Retry kéo dài làm tăng chi phí | Phân loại lỗi, cấu hình ngưỡng và cô lập bằng DLQ. |
| Kết quả so sánh thiếu công bằng | Thống nhất nghiệp vụ, ghi rõ cấu hình và chạy lặp lại. |
| Khó xác định nguyên nhân lỗi | Log có cấu trúc, correlationId và quy trình tra cứu theo đơn hàng. |
| Lộ thông tin trong log hoặc mã nguồn | Sử dụng dữ liệu giả lập, hạn chế nội dung log và không lưu thông tin xác thực trong mã nguồn. |

## 2. Tài liệu tham khảo

- [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [Amazon CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [AWS Pricing Calculator](https://calculator.aws/)
