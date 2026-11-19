---
title: "Kiến trúc giải pháp"
weight: 3
chapter: false
pre: " <b> 2.3. </b> "
---

## 1. Kiến trúc và giải pháp đề xuất

### 1.1. Phương án xử lý đồng bộ

Luồng xử lý dự kiến:

**Client → API Gateway → Lambda xử lý nghiệp vụ → DynamoDB → Phản hồi kết quả**

Hàm xử lý thực hiện các bước nghiệp vụ trong phạm vi một yêu cầu và trả kết quả sau khi hoàn tất hoặc gặp lỗi.

Phương án này được sử dụng làm cơ sở đối chiếu về thời gian phản hồi, thời gian hoàn tất và hành vi khi có lỗi.

### 1.2. Phương án xử lý theo sự kiện

Luồng xử lý logic dự kiến:

**Client → API Gateway → Lambda tiếp nhận → SQS → Lambda xử lý → DynamoDB**

Các thành phần hỗ trợ:

- **SQS dead-letter queue:** lưu thông điệp vượt quá ngưỡng xử lý lại.
- **CloudWatch:** thu thập log, metric, dashboard và cảnh báo.
- **API truy vấn trạng thái:** cung cấp tiến độ và kết quả xử lý theo mã đơn hàng.

API tiếp nhận trả về mã đơn hàng cùng phản hồi tiếp nhận sau khi yêu cầu đã được đưa vào cơ chế lưu bền vững đã thiết kế. Phản hồi này không có nghĩa là toàn bộ nghiệp vụ đã hoàn tất.

Nếu cần lưu trạng thái tiếp nhận vào DynamoDB đồng thời phát thông điệp, thiết kế phải giải quyết trường hợp một thao tác thành công nhưng thao tác còn lại thất bại. Một phương án được xem xét là transactional outbox: ghi đơn hàng và bản ghi công việc trong cùng giao dịch, sau đó chuyển công việc sang hàng đợi bằng bộ chuyển tiếp có cơ chế thử lại và đối soát.

Nếu thông điệp được đưa trực tiếp vào SQS trước khi consumer tạo bản ghi đơn hàng, API truy vấn phải mô tả rõ khoảng thời gian trạng thái chưa xuất hiện trong cơ sở dữ liệu.

Lựa chọn cuối cùng sẽ được ghi rõ trong tài liệu triển khai và kiểm thử tích hợp.

### 1.3. Vai trò của các dịch vụ AWS

| Dịch vụ | Vai trò dự kiến |
|---------|----------------|
| Amazon API Gateway | Cung cấp API tạo đơn hàng và truy vấn trạng thái. |
| AWS Lambda | Kiểm tra yêu cầu, xử lý nghiệp vụ và cập nhật trạng thái. |
| Amazon SQS | Lưu đệm công việc và tách tốc độ tiếp nhận khỏi tốc độ xử lý. |
| Amazon SQS DLQ | Cô lập thông điệp xử lý thất bại nhiều lần. |
| Amazon DynamoDB | Lưu trạng thái đơn hàng và dữ liệu phục vụ idempotency. |
| Amazon CloudWatch | Thu thập log, theo dõi metric và phát cảnh báo. |
| AWS IAM | Quản lý quyền truy cập giữa các thành phần theo nguyên tắc quyền tối thiểu. |

Các dịch vụ định tuyến hoặc thông báo bổ sung chỉ được đưa vào khi có yêu cầu cụ thể, tránh mở rộng kiến trúc vượt quá phạm vi đánh giá.
