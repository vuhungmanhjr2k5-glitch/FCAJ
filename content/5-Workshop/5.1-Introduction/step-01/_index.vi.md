---
title: "Bạn sẽ làm gì"
weight: 1
chapter: false
pre: " <b> 5.1.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

- Tách bước nhận đơn khỏi bước xử lý bằng **Amazon SQS** và hàng đợi lỗi (**DLQ**)
- Chống xử lý trùng bằng **Amazon DynamoDB** (idempotency)
- Điều phối quy trình gửi email và thông báo bằng **AWS Step Functions**, **Amazon SES**, **Amazon SNS**
- Theo dõi hệ thống và nhận cảnh báo với **Amazon CloudWatch**
- Giả lập lỗi và tải để đánh giá hiệu quả của kiến trúc

<!-- workshop-detail:v1:begin -->
## Kết quả đầu ra sau khi thực hành

Người học cần giải thích được mỗi dịch vụ giải quyết vấn đề gì, triển khai được cấu hình và kiểm chứng được kết quả bằng dữ liệu thực tế. Việc tài nguyên xuất hiện trong AWS Console là bước đầu; cần tiếp tục kiểm tra chúng phối hợp đúng với nhau.

| Công việc | Kết quả mong đợi | Cách kiểm tra |
|---|---|---|
| Tiếp nhận đơn | Yêu cầu hợp lệ được chấp nhận và có mã theo dõi | Phản hồi API hoặc giao diện website |
| Xử lý đơn | Workflow đi đến trạng thái kết thúc phù hợp | Execution và bản ghi DynamoDB |
| Kiểm soát trùng | Yêu cầu lặp được nhận biết | Log cùng mã đơn và lịch sử tác vụ |
| Xử lý thất bại | Lỗi được ghi nhận, có đường xử lý tiếp theo | Nhánh Catch, trạng thái FAILED hoặc DLQ tùy loại lỗi |
| Giám sát | Người vận hành xác định được tài nguyên và thời điểm lỗi | Dashboard, alarm history và thông báo |

## Tình huống minh họa

Một người đặt hai cốc trên website Mộc. Website trả mã đơn sau khi tiếp nhận; công việc gửi email tiếp tục ở phía sau. Nếu người dùng chưa rõ lần gửi trước có thành công hay không, lần thử lại cần giữ danh tính của yêu cầu để hạn chế tạo đơn mới ngoài ý muốn.

## Điều kiện đánh giá

Kết luận của từng phép thử phải giới hạn trong dữ liệu đã kiểm tra. Một lần gửi trùng thành công không chứng minh hệ thống bảo đảm exactly-once trong mọi tình huống. Bài thực hành cần chỉ rõ đầu vào, số lần gửi, thời điểm quan sát và kết quả thực tế.

Phần này có thể dùng bảng mục tiêu mà không cần ảnh mới. Các ảnh kiểm thử sẽ được thu thập khi thực hiện các mục triển khai và thực nghiệm tương ứng.
<!-- workshop-detail:v1:end -->

## Theo một đơn từ lúc bấm mua đến khi có kết quả

Giả sử khách đặt hai chiếc cốc và trình duyệt gửi đơn `MOC-101`. API bất đồng bộ trả mã HTTP 202 sau khi SQS chấp nhận thông điệp. Lúc này giao diện có thể hiển thị “Đã tiếp nhận MOC-101”. Hàng đợi vẫn có thể đang chờ một Lambda rảnh. Chỉ khi workflow chạy xong bước gửi thư và cập nhật bảng thì ứng dụng mới đọc được trạng thái `DONE`.

Hãy đổi một chi tiết trong tình huống trên. Kết nối của khách ngắt ngay sau khi bấm mua và khách bấm lại. Hai yêu cầu có thể tạo hai tin nhắn SQS với hai `MessageId`, trong khi chỉ có một ý định mua hàng. Giữ cùng `orderId` cho lần thử lại cho phép Lambda nhận ra quan hệ này. Nếu khách thực sự muốn mua thêm hai chiếc cốc, đơn tiếp theo phải có mã mới.

## Đọc kết quả theo câu hỏi của người dùng

| Người hỏi | Câu hỏi | Dữ liệu đủ để trả lời |
|---|---|---|
| Khách hàng | Hệ thống đã nhận đơn của tôi chưa? | Mã phản hồi và `orderId` mà API đã tiếp nhận |
| Nhân viên hỗ trợ | Đơn đang ở bước nào? | Bản ghi theo đúng khóa và lịch sử execution cùng tên |
| Người vận hành | Tôi cần xử lý lỗi ở đâu? | Bước thất bại trong workflow, hoặc body trong DLQ nếu Lambda chưa đọc được đơn |

Đến cuối bài, thử kể lại một đơn bằng ba dòng có cùng mã. Dòng đầu ghi khi hệ thống nhận yêu cầu. Dòng thứ hai ghi thành phần xử lý tiếp theo. Dòng cuối ghi kết quả quan sát được. Nếu mới thấy hai dòng đầu, kết quả của đơn vẫn đang chờ xác nhận.
