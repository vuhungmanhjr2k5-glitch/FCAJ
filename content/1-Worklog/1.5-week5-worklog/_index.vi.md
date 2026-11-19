---
title: "Worklog Tuần 5"
date: 2026-09-07
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

**Thời gian thực tập:** 07/09/2026 - 12/09/2026

### Mục tiêu tuần 5:

- Hiểu cách tách thành phần phát sự kiện khỏi thành phần xử lý.
- Xây dựng cấu trúc sự kiện và thực hành định tuyến bằng Amazon EventBridge.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 07/09/2026 | Phân tích sự khác nhau giữa gọi xử lý trực tiếp và phát sự kiện; xác định vai trò của producer và consumer. | [Event-driven architecture](https://aws.amazon.com/event-driven-architecture/) |
| 08/09/2026 | Thiết kế sự kiện đơn hàng gồm eventId, orderId, eventType, schemaVersion và thời điểm phát sinh. | Bản nháp cấu trúc sự kiện |
| 09/09/2026 | Tạo event bus và thực hành gửi sự kiện thử nghiệm lên EventBridge. | [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) |
| 11/09/2026 | Cấu hình rule theo loại sự kiện và kiểm tra việc chuyển sự kiện tới đích thử nghiệm. | [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) |
| 12/09/2026 | Kiểm tra sự kiện thiếu trường, sai loại và không khớp rule; rà soát dữ liệu cần thiết trong payload. | Ghi chú kiểm thử sự kiện |

### Kết quả đạt được tuần 5:

- Xây dựng quy ước sự kiện để các thành phần trao đổi dữ liệu nhất quán.
- Thực hành phát và định tuyến sự kiện, đồng thời nhận diện các trường hợp không được xử lý như mong đợi.