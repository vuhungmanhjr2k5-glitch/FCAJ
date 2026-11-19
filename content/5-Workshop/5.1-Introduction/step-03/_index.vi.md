---
title: "Chỉ số đánh giá"
weight: 3
chapter: false
pre: " <b> 5.1.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

- Thời gian phản hồi của API đặt hàng (trung bình, p95)
- Tỉ lệ đơn bị mất khi có lỗi
- Số email gửi trùng
- Thời gian từ lúc xảy ra lỗi đến lúc quản trị viên nhận cảnh báo

<!-- workshop-detail:v1:begin -->
## Phân biệt các loại thời gian

Thời gian phản hồi API đo từ lúc gửi yêu cầu đến lúc nhận phản hồi. Với luồng bất đồng bộ, phản hồi này có thể chỉ xác nhận đã tiếp nhận. Thời gian hoàn tất đo đến khi đơn đạt trạng thái cuối; thời gian nhận email còn bao gồm quá trình chuyển phát sau khi SES chấp nhận thư.

| Chỉ số | Ý nghĩa | Nguồn kiểm tra |
|---|---|---|
| API trung bình | Trung bình thời gian phản hồi trong lượt thử | Công cụ gửi yêu cầu |
| API P95 | Phần vị của thời gian phản hồi, thể hiện nhóm yêu cầu chậm | Cùng công cụ và phương pháp tính giữa các lượt |
| DONE / FAILED / đang xử lý | Kết quả tại thời điểm kiểm tra | DynamoDB và execution |
| Tác vụ gửi email / thư nhận được | Hai phép đếm khác nhau | Execution history / hộp thư |
| Độ trễ cảnh báo | Khoảng cách giữa mốc lỗi và mốc cảnh báo đã chọn | Execution, alarm history hoặc email |

## Cách ghi nhận một lượt thử

Ghi lại mã lượt thử, số yêu cầu, concurrency, khoảng nghỉ, thời điểm bắt đầu và thời điểm đối chiếu. So sánh hai phương án trong điều kiện tương đương và dùng cùng cách tính P95. Lượt ít mẫu chỉ phản ánh điều kiện thử đó; không đủ để kết luận khả năng chịu tải lớn.

Đơn chưa được tìm thấy tại một lần kiểm tra chưa đủ để kết luận bị mất vĩnh viễn. Cần đối chiếu phản hồi API, hàng đợi, log và kiểm tra lại sau khoảng chờ hợp lý. Với cảnh báo, mốc alarm chuyển ALARM khác với mốc email đến hộp thư.

**Mẫu ghi chép:** mã lượt thử; số yêu cầu; số phản hồi thành công; DONE; FAILED; chưa kết thúc; API trung bình; API P95; thời điểm đối chiếu. Phần này chưa cần chạy tải hoặc chụp ảnh mới.
<!-- workshop-detail:v1:end -->

## Tính P95 bằng một ví dụ nhỏ

Giả sử có 20 phản hồi HTTP. Sắp thời gian phản hồi từ ngắn đến dài rồi lấy vị trí làm tròn lên của `0,95 × 20`, tức mẫu thứ 19 nếu đếm từ 1. Với phương pháp này, một yêu cầu rất chậm ở vị trí 20 có thể làm tăng trung bình mà không làm đổi P95. Vì thế khi ghi P95 cần ghi cả số phản hồi đo được. Công cụ không có thời gian phản hồi cho yêu cầu đứt kết nối phải đếm yêu cầu đó vào lỗi mạng.

## Chọn mẫu số cho tỷ lệ hoàn tất

Ví dụ minh họa: gửi 20 mã đơn riêng biệt, nhận 20 phản hồi chấp nhận, sau một phút bảng có 17 `DONE`, 1 `FAILED` và 2 `RECEIVED`. Tỷ lệ hoàn tất thành công tại thời điểm đó là 17/20. Hai đơn `RECEIVED` vẫn truy vết được. Kiểm tra lại sau khi workflow có đủ thời gian thử lại rồi ghi thêm một dòng kết quả có thời điểm mới. Các con số trong ví dụ chỉ minh họa cách tính.

Với phép gửi trùng, đổi đơn vị đếm. Gửi năm lần cùng một `orderId` nghĩa là có năm lần nộp và một đơn nghiệp vụ. Báo cáo riêng số yêu cầu, số mã khác nhau, số execution và số lần tác vụ gửi thư thành công. Cách ghi này giúp một dòng DynamoDB duy nhất vẫn có thể bộc lộ vấn đề gửi email hai lần.
