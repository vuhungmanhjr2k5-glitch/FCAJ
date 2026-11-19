---
title: "Kiến trúc"
weight: 2
chapter: false
pre: " <b> 5.1.2 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

{{< figure src="/images/5-Workshop/5.1-Introduction/101.png" alt="Kiến trúc tổng thể" >}}

<!-- workshop-detail:v1:begin -->
## Cách đọc sơ đồ kiến trúc

Đọc sơ đồ theo đường đi của một yêu cầu. Phần tiếp nhận đưa đơn vào SQS; Lambda xử lý thông điệp và khởi động Step Functions. DynamoDB hỗ trợ kiểm soát trùng và lưu trạng thái. Step Functions gọi SES để gửi email, sau đó cập nhật kết quả hoặc đi nhánh xử lý lỗi.

| Chặng | Đường đi |
|---|---|
| Tiếp nhận | API → SQS |
| Xử lý | Lambda → Step Functions → SES |
| Lỗi khi nhận | SQS → DLQ khi vượt ngưỡng nhận |
| Dữ liệu và quan sát | DynamoDB lưu trạng thái, CloudWatch tổng hợp tín hiệu, SNS chuyển thông báo |

## Trách nhiệm của từng tầng

- **API:** kiểm tra đầu vào và tiếp nhận yêu cầu; không coi việc tiếp nhận là hoàn tất đơn.
- **SQS:** giữ thông điệp chờ xử lý và hỗ trợ giao lại khi cần.
- **Lambda:** thực hiện logic xử lý thông điệp theo mã ứng dụng.
- **Step Functions:** quản lý trình tự, các lần thử lại và nhánh bắt lỗi.
- **DynamoDB:** cung cấp trạng thái để ứng dụng tra cứu và dữ liệu phục vụ kiểm tra trùng.

## Liên hệ với website Mộc

Bản website dùng địa chỉ execute-api phục vụ giao diện qua API Gateway, Lambda đọc file và S3 riêng tư. Các yêu cầu đặt hàng đi qua Lambda API rồi vào hàng đợi của luồng nguồn. Đây là phần giao diện bổ sung; cần phân biệt Lambda phục vụ file với Lambda API và Lambda xử lý đơn.

Lỗi gửi email bên trong Step Functions không tự động có nghĩa thông điệp gốc được chuyển sang DLQ. DLQ xử lý vấn đề giao/nhận thông điệp SQS theo cấu hình; workflow có cơ chế Catch riêng.

**Minh chứng:** dùng lại sơ đồ kiến trúc đã có. Nếu sơ đồ chỉ mô tả luồng nguồn, ghi rõ phạm vi đó; không cần triển khai thêm tài nguyên để chụp một sơ đồ mới.
<!-- workshop-detail:v1:end -->

## Hai ranh giới dễ nhầm trên sơ đồ

Đầu tiên hãy đi theo mũi tên từ SQS đến Lambda. Trong thời gian Lambda xử lý, thông điệp tạm ẩn đối với consumer khác. Nếu Lambda trả về thành công, dịch vụ tích hợp SQS có thể xóa thông điệp khỏi hàng chính. Với phiên bản ở mục 5.5, Lambda hoàn thành phần việc của nó ngay sau khi khởi động thành công Step Functions. Vì vậy việc hàng chính trở về 0 có thể xảy ra trước khi khách nhận thư.

Tiếp theo đi từ Step Functions đến SES. Khi SES từ chối yêu cầu, lịch sử execution ghi lần gọi lỗi và các lần thử lại theo cấu hình. Nếu vẫn thất bại, workflow ghi `FAILED` và gửi SNS. Người vận hành dùng tên execution để tra bước lỗi. Ngược lại, một body hỏng cú pháp JSON có thể khiến Lambda dừng trước khi khởi động workflow. Khi đó điểm quan sát phù hợp là log Lambda và hàng DLQ.

## Dự đoán điểm dừng

Chọn một đơn đã gửi rồi lần lượt trả lời ba câu. API đã ghi nhận việc đưa tin vào SQS chưa? Lambda đã đọc được `orderId` và khởi động execution chưa? Execution đã đạt trạng thái cuối chưa? Lần theo thứ tự này giúp thu hẹp phạm vi tra cứu ngay cả lúc dashboard đang hiển thị số liệu tổng hợp của nhiều đơn. Hệ thống giữ địa chỉ hàng đợi và ARN của state machine trong cấu hình triển khai, nên khi đổi stack phải đọc lại các giá trị đó.
