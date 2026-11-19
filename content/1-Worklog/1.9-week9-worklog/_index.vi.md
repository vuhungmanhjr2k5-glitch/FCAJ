---
title: "Worklog Tuần 9"
date: 2026-10-05
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

**Thời gian thực tập:** 05/10/2026 - 10/10/2026

### Mục tiêu tuần 9:

- Tích hợp các thành phần thành luồng xử lý đơn hàng theo sự kiện.
- Kiểm tra sự thống nhất giữa phản hồi tiếp nhận, sự kiện và trạng thái đơn hàng.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 05/10/2026 | Rà soát sơ đồ API Gateway, Lambda, EventBridge, SQS và DynamoDB; kiểm tra quyền truy cập giữa các thành phần. | Sơ đồ kiến trúc dự án |
| 06/10/2026 | Hoàn thiện API tiếp nhận đơn hàng, trả mã đơn hàng và trạng thái tiếp nhận theo hợp đồng API đã xác định. | Đặc tả API dự án |
| 07/10/2026 | Tích hợp bước phát sự kiện và xử lý kết quả gửi; phân tích rủi ro lệch trạng thái khi ghi dữ liệu thành công nhưng phát sự kiện thất bại. | [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) |
| 09/10/2026 | Hoàn thiện consumer xử lý nghiệp vụ, cập nhật trạng thái và ghi nhận thời điểm hoàn tất hoặc thất bại. | Mã nguồn xử lý đơn hàng |
| 10/10/2026 | Kiểm thử toàn luồng từ lúc tạo đơn đến khi truy vấn kết quả; đối chiếu log, thông điệp và dữ liệu lưu trữ. | Biên bản kiểm thử tích hợp |

### Kết quả đạt được tuần 9:

- Tích hợp các thành phần của phương án xử lý theo sự kiện trong môi trường thử nghiệm.
- Xác định các điểm cần kiểm soát để tránh phản hồi tiếp nhận thành công khi đơn hàng chưa được đưa vào luồng xử lý một cách tin cậy.