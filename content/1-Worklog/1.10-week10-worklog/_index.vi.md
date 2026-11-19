---
title: "Worklog Tuần 10"
date: 2026-10-12
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

**Thời gian thực tập:** 12/10/2026 - 17/10/2026

### Mục tiêu tuần 10:

- Kiểm tra hành vi của hệ thống khi thông điệp được giao lại hoặc thành phần xử lý gặp lỗi.
- Hoàn thiện cách xử lý idempotency và quy trình phục hồi thông điệp lỗi.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 12/10/2026 | Lập ma trận lỗi gồm thông điệp trùng, dữ liệu không hợp lệ, timeout và lỗi tạm thời tại bước xử lý. | Kế hoạch kiểm thử lỗi |
| 13/10/2026 | Thiết kế khóa idempotency theo thao tác nghiệp vụ; sử dụng ghi có điều kiện để kiểm soát việc xử lý lặp và xử lý đồng thời. | [DynamoDB condition expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html) |
| 14/10/2026 | Gửi lại thông điệp và mô phỏng lỗi giữa các bước; kiểm tra tác động nghiệp vụ, trạng thái xử lý và cách tiếp tục sau lỗi. | Biên bản kiểm thử idempotency |
| 16/10/2026 | Mô phỏng lỗi tạm thời và lỗi kéo dài; quan sát retry, tồn đọng, cảnh báo và thông điệp chuyển sang dead-letter queue. | [SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) |
| 17/10/2026 | Khắc phục nguyên nhân lỗi thử nghiệm, thực hành redrive có kiểm soát và kiểm tra lại kết quả đơn hàng. | Ghi chú quy trình phục hồi |

### Kết quả đạt được tuần 10:

- Xây dựng các kịch bản kiểm tra idempotency và khả năng phục hồi khi xử lý bị gián đoạn.
- Ghi nhận hành vi thực tế theo từng kịch bản, không đồng nhất cơ chế giao ít nhất một lần với bảo đảm xử lý đúng một lần.
- Kết quả cần bổ sung: [Số trường hợp đạt/chưa đạt, lỗi còn tồn tại và phạm vi tác động nghiệp vụ đã kiểm chứng.]