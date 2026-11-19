---
title: "Chống xử lý trùng"
weight: 4
chapter: false
pre: " <b> 5.4 </b> "
---

<!-- workshop-split:v1 -->

## Mục tiêu

Hàng đợi SQS đảm bảo giao tin nhắn **ít nhất một lần** (at-least-once), nghĩa là
đôi khi cùng một đơn hàng có thể được giao hai lần. Nếu không xử lý, khách sẽ
nhận email trùng. Ta dùng **DynamoDB** làm "sổ ghi": mỗi `orderId` chỉ được xử
lý một lần. Kỹ thuật này gọi là **idempotency**.

## Các phần thực hành

Mở từng mục dưới đây hoặc dùng menu bên trái. Thực hiện theo thứ tự; các bước sau có thể dùng tài nguyên và cấu hình từ bước trước.

- [5.4.1 Cập nhật template.yaml](step-01/)
- [5.4.2 Viết mã Lambda](step-02/)
- [5.4.3 Triển khai lại](step-03/)
- [5.4.4 Kiểm thử](step-04/)
- [5.4.5 Kiểm tra kết quả](step-05/)
- [5.4.6 Xử lý sự cố](step-06/)
