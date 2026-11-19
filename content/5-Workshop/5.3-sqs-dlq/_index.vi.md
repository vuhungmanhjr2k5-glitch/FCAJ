---
title: "Hàng đợi và DLQ"
weight: 3
chapter: false
pre: " <b> 5.3 </b> "
---

<!-- workshop-split:v1 -->

## Mục tiêu

Tạo hàng đợi chính nhận đơn hàng và một hàng đợi lỗi (**Dead-Letter Queue, DLQ**)
để giữ lại các tin nhắn xử lý thất bại, thay vì để chúng bị mất.

Ba khái niệm cần nhớ:

- **Visibility timeout**: thời gian tin nhắn bị ẩn sau khi một consumer nhận nó. Nếu consumer không xóa tin nhắn trong thời gian này, tin nhắn hiện lại để được thử lần nữa.
- **maxReceiveCount**: số lần tin nhắn được nhận mà không bị xóa trước khi chuyển sang DLQ.
- **DLQ**: hàng đợi chứa các tin nhắn lỗi để kiểm tra và xử lý lại.

## Các phần thực hành

Mở từng mục dưới đây hoặc dùng menu bên trái. Thực hiện theo thứ tự; các bước sau có thể dùng tài nguyên và cấu hình từ bước trước.

- [5.3.1 Tạo file template.yaml](step-01/)
- [5.3.2 Triển khai](step-02/)
- [5.3.3 Kiểm tra trên Console](step-03/)
- [5.3.4 Thử để tin nhắn rơi vào DLQ](step-04/)
- [5.3.5 Kiểm tra kết quả](step-05/)
