---
title: "Giới thiệu và kiến trúc"
date: "2026-10-03"
weight: 1
chapter: false
pre: " <b> 5.1 </b> "
---

<!-- workshop-split:v1 -->

## Tổng quan

Trong một ứng dụng thương mại điện tử, bước đặt hàng kéo theo nhiều tác vụ phía sau: ghi nhận đơn, gửi email xác nhận, thông báo cho quản trị viên. Nếu xử lý tất cả trong cùng một yêu cầu (đồng bộ), hệ thống dễ chậm khi lượng đơn tăng, và một dịch vụ phụ bị lỗi có thể làm mất đơn hoặc gửi email trùng.

Workshop này xây dựng luồng xử lý đơn hàng theo hướng **event-driven, chịu lỗi và quan sát được** trên AWS, sau đó đo đạc để so sánh với phương án xử lý đồng bộ.

<!-- workshop-detail:v1:begin -->
## Vấn đề cần giải quyết

Trong cách xử lý đồng bộ, API có thể phải chờ ghi dữ liệu và gửi email rồi mới trả kết quả. Nếu một bước chậm hoặc lỗi, người dùng phải chờ lâu hơn hoặc nhận phản hồi lỗi. Khi người dùng gửi lại yêu cầu, ứng dụng cũng cần biết đó là đơn mới hay lần thử lại của đơn cũ.

Thiết kế bất đồng bộ tách việc tiếp nhận khỏi việc hoàn tất. API đưa yêu cầu vào hàng đợi; tiến trình xử lý phía sau tiếp tục thực hiện công việc. Cách tổ chức này đòi hỏi trạng thái đơn rõ ràng, cơ chế chống trùng và khả năng quan sát lỗi.

## Các nguyên tắc của bài thực hành

- **Theo dõi bằng mã đơn:** dùng cùng orderId để liên kết yêu cầu, dữ liệu và execution.
- **Dự kiến thông điệp được giao lại:** SQS Standard không bảo đảm chỉ giao đúng một lần.
- **Lưu trạng thái xử lý:** phản hồi tiếp nhận chưa phải kết quả hoàn tất.
- **Kiểm chứng bằng nhiều nguồn:** kết hợp log, DynamoDB, lịch sử execution và email.

Bốn mục con bên dưới lần lượt xác định việc cần làm, giải thích kiến trúc, cách đánh giá và kế hoạch thời gian/chi phí. Phần này tập trung giúp người đọc hiểu thiết kế trước khi triển khai; chưa cần tạo thêm tài nguyên hoặc chụp thêm ảnh Console.
<!-- workshop-detail:v1:end -->

## Các phần thực hành

Mở từng mục dưới đây hoặc dùng menu bên trái. Thực hiện theo thứ tự; các bước sau có thể dùng tài nguyên và cấu hình từ bước trước.

- [5.1.1 Bạn sẽ làm gì](step-01/)
- [5.1.2 Kiến trúc](step-02/)
- [5.1.3 Chỉ số đánh giá](step-03/)
- [5.1.4 Thời lượng và chi phí](step-04/)
