---
title: "Phạm vi triển khai"
weight: 2
chapter: false
pre: " <b> 2.2. </b> "
---

## 1. Phạm vi triển khai

### 1.1. Chức năng trong phạm vi

Hệ thống sử dụng dữ liệu giả lập để thực hiện các chức năng:

- Tiếp nhận yêu cầu tạo đơn hàng.
- Kiểm tra các trường dữ liệu bắt buộc.
- Thực hiện các bước nghiệp vụ mô phỏng với thời gian xử lý có thể điều chỉnh.
- Ghi nhận và cập nhật trạng thái đơn hàng.
- Cung cấp chức năng truy vấn trạng thái.
- Ghi nhận kết quả thông báo mô phỏng.
- Kiểm thử lỗi, xử lý trùng và khả năng phục hồi.

Các trạng thái dự kiến gồm:

| Trạng thái | Ý nghĩa |
|------------|---------|
| ACCEPTED | Đơn hàng đã được tiếp nhận theo cơ chế lưu bền vững được lựa chọn. |
| PROCESSING | Đơn hàng đang được xử lý. |
| COMPLETED | Các bước nghiệp vụ bắt buộc đã hoàn tất. |
| FAILED | Đơn hàng được xác định thất bại theo quy tắc xử lý. |

Thông điệp nằm trong dead-letter queue sẽ được theo dõi riêng để điều tra và phục hồi. Việc chuyển thông điệp vào hàng đợi lỗi cần được đối chiếu với trạng thái nghiệp vụ, không mặc định coi mọi trường hợp là đơn hàng đã thất bại vĩnh viễn.

### 1.2. Giới hạn phạm vi

- Không tích hợp thanh toán thật hoặc sử dụng dữ liệu khách hàng thật.
- Không xây dựng đầy đủ một nền tảng thương mại điện tử.
- Thử nghiệm trong một AWS Region với giới hạn tài nguyên được ghi nhận.
- Không đặt mục tiêu chứng minh khả năng vận hành ở quy mô sản xuất.
- Kết luận chỉ áp dụng cho phạm vi nghiệp vụ, cấu hình và mức tải đã kiểm thử.
