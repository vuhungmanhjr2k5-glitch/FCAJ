---
title: "Worklog Tuần 4"
date: 2026-08-31
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

**Thời gian thực tập:** 31/08/2026 - 05/09/2026

### Mục tiêu tuần 4:

- Xây dựng luồng xử lý đồng bộ với phạm vi nghiệp vụ đã xác định.
- Chuẩn bị một phương án đối chiếu có thể kiểm thử và đo đạc.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 31/08/2026 | Kết nối API Gateway với Lambda xử lý đơn hàng và kiểm tra quyền gọi hàm. | [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) |
| 01/09/2026 | Triển khai các bước kiểm tra dữ liệu, thực hiện nghiệp vụ thử nghiệm và cập nhật kết quả theo trình tự đồng bộ. | Đặc tả nghiệp vụ dự án |
| 02/09/2026 | Bổ sung phản hồi lỗi và kiểm tra sự nhất quán giữa kết quả API với trạng thái được lưu. | Ghi chú kiểm thử API |
| 04/09/2026 | Mô phỏng một bước xử lý chậm để quan sát thời gian chờ và hành vi khi vượt timeout. | [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) |
| 05/09/2026 | Ghi lại cấu hình, phiên bản mã nguồn và cách chạy thử để có thể tái lập phương án đồng bộ. | Ghi chú cấu hình thử nghiệm |

### Kết quả đạt được tuần 4:

- Xây dựng phiên bản đồng bộ làm cơ sở so sánh với luồng xử lý theo sự kiện.
- Xác định cách kiểm tra phản hồi, trạng thái đơn hàng và ảnh hưởng của bước xử lý chậm.