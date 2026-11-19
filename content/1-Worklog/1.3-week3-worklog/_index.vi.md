---
title: "Worklog Tuần 3"
date: 2026-08-24
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

**Thời gian thực tập:** 24/08/2026 - 29/08/2026

### Mục tiêu tuần 3:

- Thực hành triển khai logic xử lý bằng AWS Lambda.
- Thiết kế dữ liệu đơn hàng và lưu trữ trạng thái bằng Amazon DynamoDB.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 24/08/2026 | Tìm hiểu handler, execution role, timeout và cấu hình bộ nhớ của Lambda. | [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) |
| 25/08/2026 | Viết hàm kiểm tra dữ liệu đầu vào và tạo mã định danh cho đơn hàng thử nghiệm. | [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) |
| 26/08/2026 | Thiết kế bảng DynamoDB theo nhu cầu truy vấn đơn hàng bằng mã định danh. | [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) |
| 28/08/2026 | Tích hợp Lambda với DynamoDB để ghi nhận đơn hàng và cập nhật trạng thái xử lý. | [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) |
| 29/08/2026 | Kiểm tra thao tác đọc, ghi và cập nhật; rà soát quyền truy cập và phản hồi khi không tìm thấy đơn hàng. | Ghi chú kiểm thử chức năng |

### Kết quả đạt được tuần 3:

- Xây dựng các hàm cơ bản phục vụ tiếp nhận và truy vấn đơn hàng.
- Hình thành mô hình dữ liệu lưu trạng thái và các mốc thời gian cần thiết cho việc theo dõi xử lý.