---
title: "Kết luận"
weight: 8
chapter: false
pre: " <b> 5.6.8 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Bộ minh chứng ghi nhận dashboard, alarm Step Functions/Lambda, thông điệp thử trong DLQ và email ALARM của cả ba alarm Step Functions, Lambda, DLQ. Cảnh báo DLQ phản ánh thông điệp tồn đọng trước lần thử. Bộ ảnh chưa đủ để kết luận phục hồi sau lỗi hoặc tính độ trễ cảnh báo từ thời điểm lỗi thực tế.

## Trả lời ba câu hỏi của người trực hệ thống

**Có sự cố ở tầng nào?** Chọn thời gian trong dashboard, nhìn nhóm Lambda và Step Functions trước để biết lỗi xảy ra lúc đọc tin hay sau khi đã bắt đầu workflow. Kiểm tra cả độ sâu và tuổi hàng đợi để phân biệt một lỗi đầu vào riêng lẻ với một khoảng nhiều đơn bị chờ.

**Đơn nào bị ảnh hưởng?** Từ thời gian trên metric đi đến log hoặc execution, tìm `orderId` hoặc dấu hiệu body của lần thử. Nếu body không phải JSON, không cố tìm một item theo mã đơn chưa từng được đọc. Tên queue và nội dung trong DLQ có thể nối trực tiếp đến trường hợp đó.

**Người trực đã được báo chưa?** Xem CloudWatch History để xác định có đổi trạng thái, sau đó tìm thư có đúng tên alarm. Nếu topic cũng nhận publish từ workflow, đọc subject và nội dung trước khi gán thư cho alarm. Một email chứa mã đơn và một email chứa datapoint có thể phát sinh từ cùng một sự cố nhưng đại diện cho hai kênh báo khác nhau.

## Bàn giao trước khi sang phép so sánh API

Ghi tên stack ứng dụng, stack giám sát, Region và tên ba alarm. Đánh dấu alarm nào còn ALARM, DLQ còn tin nào và kết quả nào chưa đủ mốc thời gian. Ở phần 5.7, hai API dùng lại hàng đợi, bảng và workflow nên một sự cố đang tồn tại ở đây có thể ảnh hưởng kết quả đo sau. Chỉ bắt đầu bộ đo với mốc nền đã biết. Có thể bàn giao trọn phần giám sát bằng bảng giá trị cùng một đoạn diễn giải.
