---
title: "Thời lượng và chi phí"
weight: 4
chapter: false
pre: " <b> 5.1.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

<!-- workshop-detail:v1:begin -->
## Phân bổ thời gian thực hành

Bảng dưới đây là gợi ý lập kế hoạch, không phải thời gian đã đo. Thời gian thực tế phụ thuộc việc công cụ đã được cài đặt, quyền AWS có đầy đủ và người học có phải xử lý lỗi môi trường hay không.

| Nhóm công việc | Khoảng thời gian tham khảo |
|---|---|
| Đọc kiến trúc, kiểm tra công cụ và danh tính | 30–45 phút |
| Triển khai hàng đợi, xử lý trùng và điều phối | 60–90 phút |
| Kiểm tra lỗi, dashboard và cảnh báo | 30–60 phút |
| Tổng hợp minh chứng và dọn dẹp | 20–30 phút |

## Thành phần có thể phát sinh chi phí

Chi phí phụ thuộc lượt gọi API, số lần Lambda chạy và thời lượng thực thi, yêu cầu DynamoDB/SQS, state transition của Step Functions, email SES, log và các thành phần CloudWatch. Nếu triển khai website, cần tính thêm tài nguyên phục vụ giao diện như S3 và Lambda/API Gateway.

Không mặc định toàn bộ bài lab được miễn phí. Quyền lợi Free Tier hoặc credit phụ thuộc tài khoản và chương trình đang áp dụng. Cost Explorer hiển thị số đã làm tròn hoặc chưa cập nhật đủ không chứng minh chi phí cuối cùng bằng không.

## Cách kiểm soát

Thực hiện lượt thử nhỏ trước, tránh chạy vòng lặp gửi liên tục, đặt Budget và lưu tên các stack đã tạo. Sau khi lưu minh chứng và không còn cần chạy demo, thực hiện mục dọn dẹp. Các tài nguyên có Retain hoặc được tạo ngoài stack cần được rà soát riêng.

Xem minh chứng Budget tại mục 5.2.5. Mức 5 USD của order-workshop-budget là ngân sách đã cấu hình, không phải chi phí thực tế đã phát sinh.
<!-- workshop-detail:v1:end -->

## Chọn điểm dừng giữa hai buổi

Nếu chia workshop thành hai buổi, hãy dừng sau một stack đã triển khai thành công và một kết quả có thể đọc lại. Chẳng hạn, sau mục 5.3 hãy ghi tên stack, Region, hai tên hàng đợi và mã thông điệp thử. Buổi sau có thể đọc lại Outputs trong CloudFormation rồi tiếp tục thêm Lambda. Dừng giữa lúc stack đang rollback sẽ làm bước đầu buổi sau trở thành xử lý sự cố.

Phần thử lỗi cần nhiều thời gian chờ hơn thao tác gửi. Ở mục 5.3, thông điệp có thể ẩn trong một khoảng visibility timeout. Ở mục 5.5, workflow có thể đang chờ lần gọi SES kế tiếp. Ở mục 5.6, CloudWatch còn phải nhận datapoint và đánh giá alarm. Trong các khoảng này, có thể điền mã đơn và mốc UTC vào ghi chép thay cho việc gửi thêm yêu cầu.

## Ước lượng từ số lần thực hiện

Một lần gọi API có thể kéo theo một lần gửi SQS, một lượt Lambda và nhiều bước Step Functions. Khi một bước thử lại, số tác vụ có thể tăng dù số đơn giữ nguyên. Trong sổ chi phí, ghi số đơn riêng biệt, số lần gửi, số lần thử lại và khoảng thời gian giữ log. Những đơn vị này giúp giải thích biến động khi xem hóa đơn theo dịch vụ. Trước lúc rời buổi thực hành, ghi rõ stack nào còn phục vụ buổi sau và stack nào đã hoàn tất mục đích.
