---
title: "Worklog Tuần 11"
date: 2026-10-19
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

**Thời gian thực tập:** 19/10/2026 - 24/10/2026

### Mục tiêu tuần 11:

- Thu thập dữ liệu của hai phương án theo cùng bộ kịch bản thử nghiệm.
- Phân tích hiệu năng và hành vi khi có lỗi dựa trên số liệu đo được.

### Các công việc đã thực hiện trong tuần:

| Ngày | Công việc | Tài liệu tham khảo |
|------|-----------|--------------------|
| 19/10/2026 | Chốt phiên bản mã nguồn, cấu hình tài nguyên, dữ liệu đầu vào và giới hạn tải; ghi rõ khác biệt cấu hình giữa hai phương án. | Hồ sơ cấu hình thử nghiệm |
| 20/10/2026 | Chạy kịch bản tải ổn định cho phương án đồng bộ và event-driven; lưu dữ liệu thô theo từng lần chạy. | Công cụ và kịch bản kiểm thử |
| 21/10/2026 | Chạy kịch bản tải tăng đột ngột và bước xử lý chậm; theo dõi lỗi, tồn đọng và thời gian xử lý hết hàng đợi. | Dashboard giám sát thử nghiệm |
| 23/10/2026 | Tổng hợp p50, p95 của thời gian phản hồi và hoàn tất; tính thông lượng theo đơn hàng duy nhất, đồng thời thống kê đơn lỗi và chưa hoàn tất. | Dữ liệu đo và bảng tính kết quả |
| 24/10/2026 | Đối chiếu kết quả giữa các lần chạy; phân tích sự đánh đổi giữa độ trễ, khả năng hấp thụ tải và độ phức tạp vận hành. | Bản nháp phân tích kết quả |

### Kết quả đạt được tuần 11:

- Xây dựng bảng so sánh tách riêng độ trễ phản hồi API và thời gian hoàn tất đơn hàng.
- Tổng hợp các yếu tố có thể ảnh hưởng đến kết quả, gồm cấu hình tài nguyên, khởi động hàm, retry và thời gian chờ trong hàng đợi.
- Số liệu cần bổ sung: [Mức tải; số lần chạy; số đơn hoàn tất/lỗi/chưa hoàn tất; p50/p95; thông lượng và thời gian phục hồi của từng phương án.]