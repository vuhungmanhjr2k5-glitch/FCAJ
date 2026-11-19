---
title: "Tổng quan và mục tiêu"
weight: 1
chapter: false
pre: " <b> 2.1. </b> "
---

## 1. Tên đề tài

**Xây dựng luồng xử lý đơn hàng theo kiến trúc event-driven, có khả năng chịu lỗi và quan sát trên AWS; đánh giá so sánh với phương án xử lý đồng bộ.**

## 2. Bối cảnh và vấn đề cần giải quyết

Trong một ứng dụng thương mại điện tử, việc tiếp nhận đơn hàng có thể kéo theo nhiều bước xử lý như kiểm tra dữ liệu, ghi nhận đơn hàng, cập nhật trạng thái và gửi thông báo.

Với phương án đồng bộ, người dùng phải chờ các bước nằm trong luồng yêu cầu hoàn tất trước khi nhận kết quả. Khi một bước xử lý chậm hoặc dịch vụ phụ thuộc gặp sự cố, thời gian phản hồi có thể tăng và yêu cầu có thể vượt quá thời gian chờ.

Việc người dùng gửi lại yêu cầu sau khi gặp timeout cũng tạo ra nguy cơ xử lý trùng nếu hệ thống chưa có cơ chế kiểm soát phù hợp. Tuy nhiên, timeout không đồng nghĩa với việc đơn hàng chắc chắn chưa được xử lý, vì một số bước có thể đã hoàn thành trước khi kết nối bị gián đoạn.

Kiến trúc event-driven cho phép tách bước tiếp nhận đơn hàng khỏi các bước xử lý phía sau thông qua sự kiện hoặc thông điệp. Cách tiếp cận này có thể giúp hệ thống tiếp nhận yêu cầu khi thành phần xử lý đang chậm và thử lại các công việc gặp lỗi tạm thời.

Đổi lại, hệ thống cần xử lý thêm các vấn đề như thông điệp trùng, dữ liệu cập nhật không tức thời, tồn đọng hàng đợi và khó theo dõi luồng xử lý qua nhiều thành phần.

Đề tài được đề xuất nhằm xây dựng hai phương án có cùng phạm vi nghiệp vụ, từ đó đánh giá sự khác biệt bằng các thử nghiệm có kiểm soát.

## 3. Mục tiêu đề tài

### 3.1. Mục tiêu tổng quát

Xây dựng mô hình xử lý đơn hàng trên AWS theo kiến trúc event-driven, có cơ chế xử lý lỗi và theo dõi trạng thái, đồng thời so sánh với phương án đồng bộ dựa trên dữ liệu thực nghiệm.

### 3.2. Mục tiêu cụ thể

- Xây dựng API tiếp nhận đơn hàng và API truy vấn trạng thái.
- Triển khai phương án đồng bộ làm cơ sở đối chiếu.
- Triển khai phương án bất đồng bộ sử dụng hàng đợi và thành phần xử lý độc lập.
- Thiết kế cơ chế retry, dead-letter queue và phục hồi thông điệp lỗi.
- Kiểm soát tác động của yêu cầu hoặc thông điệp được gửi lại bằng idempotency.
- Liên kết log giữa các thành phần để theo dõi quá trình xử lý một đơn hàng.
- Xây dựng dashboard và cảnh báo cho lỗi xử lý, tồn đọng và thông điệp thất bại.
- Đo đạc độ trễ, thông lượng hoàn tất, tỷ lệ thành công và hành vi phục hồi của hai phương án.
- Tổng hợp điều kiện phù hợp, hạn chế và sự đánh đổi của từng kiến trúc.
