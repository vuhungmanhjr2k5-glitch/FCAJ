---
title: "Khả năng chịu lỗi và quan sát"
weight: 4
chapter: false
pre: " <b> 2.4. </b> "
---

## 1. Thiết kế khả năng chịu lỗi và quan sát

### 1.1. Retry và dead-letter queue

Hệ thống phân biệt lỗi tạm thời với lỗi dữ liệu hoặc lỗi nghiệp vụ không thể khắc phục bằng cách thử lại.

Các nội dung cần cấu hình và kiểm tra gồm:

- Timeout của hàm xử lý.
- Visibility timeout của hàng đợi.
- Kích thước batch và giới hạn xử lý đồng thời.
- Ngưỡng số lần nhận thông điệp trước khi chuyển vào DLQ.
- Báo cáo lỗi từng thông điệp trong batch.
- Quy trình kiểm tra, khắc phục nguyên nhân và redrive thông điệp lỗi.

Thông điệp không được đưa trở lại luồng xử lý một cách tự động, không giới hạn khi nguyên nhân lỗi chưa được xác định.

### 1.2. Idempotency và tính nhất quán

SQS Standard có thể giao thông điệp nhiều lần và không bảo đảm thứ tự tuyệt đối. Vì vậy, consumer cần được thiết kế để xử lý việc giao lại mà không tạo thêm tác động nghiệp vụ ngoài ý muốn.

Giải pháp dự kiến:

- Sử dụng khóa idempotency ổn định theo đơn hàng và thao tác nghiệp vụ.
- Dùng ghi có điều kiện để kiểm soát xử lý đồng thời.
- Phân biệt công việc đang xử lý với công việc đã hoàn tất.
- Xác định cách tiếp tục khi hàm bị gián đoạn giữa các bước.
- Kiểm soát chuyển trạng thái để thông điệp cũ không ghi đè trạng thái mới.
- Kiểm thử cả trường hợp thông điệp trùng và trường hợp lỗi xảy ra sau khi một tác động nghiệp vụ đã được thực hiện.

Việc lưu một khóa chống trùng không tự động bảo đảm mọi tác động bên ngoài chỉ xảy ra đúng một lần. Những giới hạn này phải được nêu rõ trong báo cáo.

### 1.3. Log, metric và cảnh báo

Log dự kiến sử dụng định dạng có cấu trúc với các trường:

- orderId.
- eventId hoặc messageId.
- correlationId.
- Tên bước xử lý.
- Trạng thái và loại lỗi.
- Thời điểm bắt đầu, kết thúc và thời gian xử lý.

Dashboard tập trung vào:

- Số yêu cầu tiếp nhận và số đơn hàng hoàn tất.
- Lambda errors, duration và throttles.
- Số thông điệp tồn đọng và tuổi thông điệp lâu nhất.
- Số thông điệp trong DLQ.
- Thời gian hoàn tất đơn hàng và tỷ lệ lỗi.

Các số liệu hàng đợi của SQS được sử dụng để giám sát xu hướng; việc đối soát số đơn hàng chính xác dựa trên mã định danh và dữ liệu thử nghiệm.
