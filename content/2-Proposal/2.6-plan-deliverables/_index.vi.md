---
title: "Kế hoạch và sản phẩm bàn giao"
weight: 6
chapter: false
pre: " <b> 2.6. </b> "
---

## 1. Kế hoạch thực hiện

Kế hoạch triển khai tập trung trong bốn tuần cuối, sử dụng kiến thức và các thành phần thử nghiệm đã chuẩn bị ở giai đoạn trước.

| Thời gian | Nội dung chính | Sản phẩm dự kiến |
|-----------|----------------|------------------|
| Tuần 9: 05/10–10/10/2026 | Chốt kiến trúc, tích hợp API, hàng đợi, consumer và lưu trữ trạng thái. | Hai luồng xử lý có thể chạy thử; sơ đồ kiến trúc và đặc tả API. |
| Tuần 10: 12/10–17/10/2026 | Hoàn thiện idempotency, retry, DLQ, log và cảnh báo; kiểm thử lỗi. | Biên bản kiểm thử lỗi và hướng dẫn phục hồi. |
| Tuần 11: 19/10–24/10/2026 | Chạy các kịch bản tải, thu thập dữ liệu và so sánh hai phương án. | Dữ liệu thô, bảng tổng hợp và biểu đồ đánh giá. |
| Tuần 12: 26/10–31/10/2026 | Rà soát kết quả, hoàn thiện tài liệu, minh chứng và báo cáo. | Bộ mã nguồn, hướng dẫn triển khai và báo cáo cuối cùng. |

## 2. Kết quả kỳ vọng và tiêu chí nghiệm thu

### 2.1. Sản phẩm bàn giao

- Mã nguồn của phương án đồng bộ và phương án event-driven.
- Sơ đồ kiến trúc và tài liệu mô tả luồng xử lý.
- Hướng dẫn triển khai, cấu hình và dọn dẹp tài nguyên.
- Bộ dữ liệu, công cụ và kịch bản thử nghiệm.
- Dashboard, truy vấn log và cấu hình cảnh báo.
- Biên bản kiểm thử retry, DLQ và idempotency.
- Báo cáo so sánh có dữ liệu và minh chứng.
- Worklog và tài liệu tổng hợp bằng tiếng Việt và tiếng Anh.

### 2.2. Tiêu chí nghiệm thu

- Cả hai phương án thực hiện đúng phạm vi nghiệp vụ đã thống nhất.
- Có thể tra cứu trạng thái và theo dõi quá trình xử lý theo mã đơn hàng.
- Các tình huống lỗi được kiểm tra bằng kịch bản có thể tái lập.
- Quy trình phục hồi thông điệp lỗi được mô tả và thực hành.
- Cơ chế chống xử lý trùng được kiểm chứng trong phạm vi tác động nghiệp vụ đã xác định.
- Kết quả so sánh tách biệt thời gian phản hồi với thời gian hoàn tất.
- Các hạn chế, trường hợp chưa đạt và vấn đề còn tồn tại được ghi rõ.

Đề tài không đặt trước kết luận rằng event-driven luôn tốt hơn xử lý đồng bộ. Kết quả kỳ vọng là xác định được điều kiện phù hợp và sự đánh đổi của từng phương án dựa trên số liệu.
