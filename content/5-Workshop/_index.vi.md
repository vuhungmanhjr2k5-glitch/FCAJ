---
title : "Workshop"
date : "2026-10-03"
weight : 5
chapter : true
pre : " <b> 5. </b> "
---

<!-- workshop-detail:v1:begin -->
<!-- moc:tong-quan -->
# Từ đơn hàng trên Mộc đến email xác nhận {#tổng-quan-workshop}

Một khách hàng chọn hai chiếc cốc trên website **Mộc**, nhập email rồi bấm **Đặt hàng**. Khi màn hình hiện mã đơn, khách có thể đóng trình duyệt. Trên AWS, hệ thống vẫn tiếp tục xử lý đơn và gửi thư xác nhận. Trong workshop này, bạn sẽ xây dựng luồng xử lý bất đồng bộ đó và dùng mã `orderId` để theo dõi từng bước.

## Một đơn hàng đi qua hệ thống như thế nào?

1. **Tiếp nhận.** API kiểm tra yêu cầu, đưa đơn vào hàng đợi Amazon SQS rồi trả về mã HTTP `202`. Lúc này, Mộc đã nhận đơn và khách có mã để tra cứu.
2. **Ghi nhận.** AWS Lambda đọc thông điệp rồi ghi đơn vào Amazon DynamoDB theo `orderId`. Khi SQS giao lại cùng thông điệp, Lambda dùng mã này để nhận ra đơn đã có trong bảng.
3. **Gửi thư.** AWS Step Functions điều phối bước gửi email qua Amazon SES. Khi SES chấp nhận yêu cầu gửi thư, quy trình ghi trạng thái `DONE`. Nếu bước gửi vẫn lỗi sau các lần thử lại, quy trình ghi `FAILED` và báo cho người vận hành qua Amazon SNS.

## Nếu khách bấm đặt hàng lần thứ hai

Giả sử kết nối bị ngắt ngay sau lần bấm đầu tiên. Trong bài kiểm thử, bạn gửi lại cùng một `orderId`, tra bản ghi DynamoDB rồi mở lịch sử thực thi của Step Functions. So sánh số yêu cầu API đã nhận với số lần bước gửi thư thành công để biết lần bấm thứ hai đã được xử lý ra sao.

Khi nhiều đơn đến cùng lúc, bạn sẽ đo riêng **thời gian API phản hồi** và **thời gian đơn đạt trạng thái cuối**. Nếu khoảng cách giữa hai mốc tăng dần, hãy xem biểu đồ hàng đợi SQS trong Amazon CloudWatch để biết có bao nhiêu thông điệp đang chờ Lambda xử lý.

{{< moc-tong-quan "style" >}}
<!-- /moc:tong-quan -->

### Những nội dung chính

| Nhóm nội dung | Vai trò trong hệ thống |
|---|---|
| SQS và DLQ | Lưu yêu cầu chờ xử lý; giữ thông điệp không xử lý được theo chính sách chuyển lỗi |
| Lambda và DynamoDB | Xử lý yêu cầu, kiểm soát trùng và lưu trạng thái đơn |
| Step Functions | Điều phối các bước và thực hiện Retry/Catch theo cấu hình |
| SES và SNS | Gửi xác nhận cho người đặt hàng và thông báo cho quản trị viên |
| CloudWatch | Theo dõi log, metric, dashboard và alarm |

### Sản phẩm và kết quả cần kiểm chứng

Website Mộc là cửa hàng thử nghiệm, chưa có thanh toán, kiểm kho hoặc giao hàng thực tế. Một đơn thử cần được đối chiếu bằng cùng mã đơn trên giao diện, DynamoDB, execution Step Functions và email. Trạng thái DONE cho biết quy trình ứng dụng hoàn tất; việc email có vào Inbox cần kiểm tra riêng.

Các phần thực hành đi từ chuẩn bị môi trường đến kiểm thử thành công, gửi trùng, gây lỗi có kiểm soát và dọn dẹp. Khi ghi báo cáo, phân biệt kết quả quan sát được với mục tiêu dự kiến; không dùng một ảnh triển khai thành công để thay cho bằng chứng cả luồng đã chạy thành công.

**Ảnh minh họa tùy chọn:** mở trang sản phẩm Mộc và chụp giao diện. Không cần gửi thêm đơn chỉ để chụp ảnh tổng quan. Ghi chú đây là giao diện minh họa; kết quả xử lý được trình bày ở phần thực nghiệm.
<!-- workshop-detail:v1:end -->

<!-- workshop-split:v1 -->

## Lộ trình thực hành

- [5.1 Giới thiệu và kiến trúc](5.1-Introduction/)
- [5.2 Chuẩn bị (IAM, region, công cụ)](5.2-Prerequisite/)
- [5.3 Hàng đợi và DLQ](5.3-sqs-dlq/)
- [5.4 Chống xử lý trùng](5.4-Idempotency/)
- [5.5 Điều phối và thông báo](5.5-Orchestration/)
- [5.6 Giám sát](5.6-Monitoring/)
- [5.7 Thực nghiệm lỗi và tải](5.7-Experiment/)
- [5.8 Clean up](5.8-Clean-Up/)
