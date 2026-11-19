---
title: "Xử lý sự cố"
weight: 6
chapter: false
pre: " <b> 5.5.6 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

- **Không nhận được email xác nhận đơn**: kiểm tra `SenderEmail` và email khách đều ở trạng thái *Verified* trong SES, và xem thư mục spam.
- **Không nhận được cảnh báo**: kiểm tra đã bấm *Confirm subscription* ở SNS chưa (Console → SNS → Subscriptions, trạng thái phải là *Confirmed*).
- **Không có execution nào**: xem log của `ProcessOrderFn`; nếu có lỗi quyền, kiểm tra phần `Policies` trong template.
- **Execution báo lỗi `AccessDenied` ở SES**: kiểm tra role của state machine có `ses:SendEmail`.

## Đi từ triệu chứng đến dịch vụ chịu trách nhiệm

Nếu khách báo thiếu email, hỏi mã đơn rồi tra bảng. Item chưa xuất hiện dẫn về hàng đợi và Lambda. Item `RECEIVED` dẫn đến việc tra xem đã có execution cùng tên chưa. Khi tìm được execution, mở ngay state cuối và lỗi đầu tiên. Cách đi này giúp tránh sửa xác minh SES trong tình huống worker còn chưa khởi động được state machine.

Nếu tác vụ SES báo `AccessDenied`, kiểm tra role của state machine và action nó được phép gọi. Nếu SES báo identity chưa xác minh, kiểm tra địa chỉ gửi, người nhận thử và Region. Nếu nhiều lần gửi liên tiếp bị giới hạn tốc độ, quan sát lịch sử retry trước khi gửi lại bằng tay. Một lần gửi bằng tay tạo thêm yêu cầu vào bài thử và có thể làm chậm việc xác định nguyên nhân ban đầu.

## Khi hệ thống ghi FAILED nhưng quản trị viên chưa nhận thư

Tìm trạng thái `NotifyAdminFailure` trong history. Nếu chưa tới trạng thái này, xem trạng thái cập nhật bảng ngay trước đó. Nếu publish bị từ chối, xác định role và ARN topic từ thông báo. Nếu publish thành công, kiểm tra subscription đã xác nhận, hộp thư đến và thư rác của đúng địa chỉ quản trị. Ghi rõ bước cuối có kết quả thay vì quy chung toàn bộ sự cố cho SNS.

Khi sửa xong, gửi một đơn thử mới với điều kiện đã sửa. Giữ đơn cũ làm mốc phân tích để so sánh. Execution Standard đã kết thúc vẫn lưu lịch sử cho lần chạy đó. Ghi mã cũ, mã mới và thay đổi duy nhất giữa hai lần là đủ cho phần giải trình.
