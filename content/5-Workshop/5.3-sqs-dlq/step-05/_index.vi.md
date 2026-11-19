---
title: "Kiểm tra kết quả"
weight: 5
chapter: false
pre: " <b> 5.3.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

- Có hai hàng đợi, hàng chính trỏ tới DLQ với `maxReceiveCount = 3`.
- Tin nhắn bị nhận quá 3 lần mà không xóa nằm trong DLQ.

## Ghép kết quả cấu hình với kết quả giao tin

Tách phần nghiệm thu thành hai dòng. Dòng cấu hình ghi ARN đích trong redrive policy, ARN của DLQ thực và giá trị `maxReceiveCount`. Dòng hành vi ghi mã body thử, ít nhất một lần thấy body ở hàng chính và nơi tìm thấy cùng body ở cuối phép thử. Người đọc có thể dựa vào đó để phân biệt “đã thiết lập đường chuyển lỗi” với “đã quan sát một tin đi qua đường đó”.

| Quan sát tại thời điểm kiểm tra | Cách diễn giải | Điều cần xem tiếp |
|---|---|---|
| Đích đúng, thấy đúng body trong DLQ | Đã quan sát được cả cấu hình và chuyển tin của lượt này | Ghi thời điểm, kết thúc bài thử |
| Đích đúng, bộ đếm DLQ tăng nhưng chưa thấy body | Có thông điệp chờ trong hàng lỗi | Dùng dấu nhận diện của lượt thử để tra |
| Hàng chính còn tin đang ẩn | Có lần nhận chưa hết visibility timeout | Đợi hết thời gian ẩn rồi đọc lại |
| Không có lần nhận thành công nào | Chưa tạo được điều kiện vượt ngưỡng nhận | Xác định đúng URL nguồn và worker đang hoạt động |

## Chuyển sang phần Lambda

Ghi rõ hàng nào còn thông điệp thử. Khi bật trigger, Lambda có thể nhận ngay tin còn ở hàng chính, kể cả body dạng văn bản do phần này tạo. Lỗi parse xuất hiện ngay sau deploy có thể liên quan đến chính tin đó. Ghi thời gian và body ban đầu sẽ giúp phân biệt điều này với lỗi của đơn mới ở mục 5.4.4. Tin đã nằm trong DLQ sẽ ở đó cho đến khi được xử lý, hết hạn hoặc bị xóa theo vòng đời của bài lab.
