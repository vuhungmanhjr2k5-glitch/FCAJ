---
title: "Worklog Tuần 2"
date: 2026-08-17
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

**Thời gian thực tập:** 17/08/2026 - 22/08/2026

### Mục tiêu tuần 2:

- Hiểu mô hình request-response và ảnh hưởng của các bước xử lý nối tiếp đến thời gian phản hồi.
- Thiết kế giao diện API và quy ước trạng thái đơn hàng dùng chung cho dự án.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 17/08/2026 | Tìm hiểu mô hình request-response, HTTP status code và các tình huống timeout khi xử lý yêu cầu. | [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) |
| 18/08/2026 | Thiết kế cấu trúc yêu cầu tạo đơn hàng, quy tắc kiểm tra dữ liệu và định dạng phản hồi lỗi. | Bản nháp đặc tả API |
| 19/08/2026 | Xác định mã định danh đơn hàng và các trạng thái tiếp nhận, đang xử lý, hoàn tất, thất bại. | Ghi chú mô hình trạng thái |
| 21/08/2026 | Thực hành tạo endpoint thử nghiệm trên API Gateway và gửi các yêu cầu hợp lệ, không hợp lệ. | [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) |
| 22/08/2026 | Hoàn thiện sơ đồ tuần tự cho luồng đồng bộ và xác định vị trí ghi nhận thời gian xử lý. | Sơ đồ luồng xử lý dự thảo |

### Kết quả đạt được tuần 2:

- Xây dựng đặc tả ban đầu cho API tạo đơn hàng và truy vấn trạng thái.
- Xác định các bước của luồng đồng bộ cùng những tình huống lỗi cần xử lý.