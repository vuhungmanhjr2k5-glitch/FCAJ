---
title: "Worklog Tuần 8"
date: 2026-09-28
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

**Thời gian thực tập:** 28/09/2026 - 03/10/2026

### Mục tiêu tuần 8:

- Xây dựng phương pháp so sánh hai kiến trúc trên cùng phạm vi nghiệp vụ.
- Chuẩn bị dữ liệu, kịch bản tạo tải và cách ghi nhận kết quả.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 28/09/2026 | Xác định các chỉ số đánh giá: độ trễ phản hồi API, thời gian hoàn tất đơn hàng, thông lượng hoàn tất và tỷ lệ thành công. | Đề cương đánh giá thử nghiệm |
| 29/09/2026 | Chuẩn bị dữ liệu đầu vào dùng chung và quy định cách đếm đơn hàng duy nhất để tránh tính retry thành đơn mới. | Bộ dữ liệu thử nghiệm |
| 30/09/2026 | Thiết kế các kịch bản tải ổn định, tải tăng đột ngột và thành phần xử lý chậm; xác định thời gian chạy và số lần lặp. | Kế hoạch kiểm thử tải |
| 02/10/2026 | Xây dựng công cụ gửi yêu cầu và ghi nhận các mốc thời gian theo mã đơn hàng; kiểm tra tính nhất quán của cách đo. | Mã nguồn công cụ kiểm thử |
| 03/10/2026 | Chạy thử với tải nhỏ; rà soát cấu hình tài nguyên, thời gian khởi động và thời gian chờ xử lý hết hàng đợi. | Ghi chú chạy thử |

### Kết quả đạt được tuần 8:

- Chuẩn bị phương pháp đo phân biệt rõ việc tiếp nhận yêu cầu với việc hoàn tất nghiệp vụ.
- Xây dựng bộ kịch bản thử nghiệm và quy tắc lưu dữ liệu phục vụ so sánh có thể tái lập.