---
title: "Phương pháp thử nghiệm"
weight: 5
chapter: false
pre: " <b> 2.5. </b> "
---

## 1. Phương pháp thử nghiệm và đánh giá

### 1.1. Nguyên tắc so sánh

Hai phương án sử dụng cùng phạm vi nghiệp vụ, cùng bộ dữ liệu đầu vào và cùng định nghĩa về một đơn hàng hoàn tất.

Cấu hình tài nguyên, giới hạn đồng thời, chính sách retry, phiên bản mã nguồn và điều kiện chạy được ghi lại để có thể tái lập thử nghiệm.

Cần phân biệt hai loại thời gian:

- **Thời gian phản hồi API:** từ lúc client gửi yêu cầu đến khi nhận phản hồi.
- **Thời gian hoàn tất đơn hàng:** từ lúc gửi yêu cầu ban đầu đến khi đơn hàng đạt trạng thái kết thúc được xác định.

Phản hồi tiếp nhận nhanh của phương án bất đồng bộ không được sử dụng thay cho thời gian hoàn tất để kết luận hệ thống xử lý nghiệp vụ nhanh hơn.

### 1.2. Các kịch bản thử nghiệm

| Kịch bản | Nội dung đánh giá |
|----------|------------------|
| Tải ổn định | Độ trễ và thông lượng khi tốc độ gửi yêu cầu ít biến động. |
| Tải tăng đột ngột | Khả năng tiếp nhận, mức tồn đọng và thời gian xử lý hết hàng đợi. |
| Một bước xử lý chậm | Ảnh hưởng đến phản hồi API và thời gian hoàn tất. |
| Lỗi tạm thời | Hành vi retry và khả năng hoàn tất sau khi lỗi được gỡ bỏ. |
| Lỗi kéo dài | Cách cô lập thông điệp, cảnh báo và phục hồi bằng DLQ. |
| Gửi lại yêu cầu hoặc thông điệp | Hiệu quả kiểm soát tác động nghiệp vụ trùng. |

### 1.3. Chỉ số đánh giá

- p50 và p95 của thời gian phản hồi API.
- p50 và p95 của thời gian hoàn tất đối với các đơn đã hoàn tất.
- Số đơn hàng duy nhất hoàn tất trong một đơn vị thời gian.
- Tỷ lệ hoàn tất, thất bại và chưa hoàn tất tại thời điểm kết thúc quan sát.
- Mức tồn đọng và thời gian xử lý hết hàng đợi.
- Thời gian phục hồi sau khi nguyên nhân lỗi được loại bỏ.
- Số tác động nghiệp vụ trùng được phát hiện.
- Chi phí thử nghiệm nếu có dữ liệu đủ để phân bổ.

Các đơn thất bại hoặc chưa hoàn tất được báo cáo riêng, tránh loại bỏ chúng khỏi phân tích và tạo ra kết luận sai lệch.

Mỗi kịch bản được chạy lặp lại. Báo cáo sẽ ghi số lần chạy, thời lượng, mức tải và sự biến động giữa các lần đo.
