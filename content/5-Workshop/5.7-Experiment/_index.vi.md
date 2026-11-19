---
title: "Thực nghiệm lỗi và tải"
weight: 7
chapter: false
pre: " <b> 5.7 </b> "
---

<!-- workshop-split:v1 -->

## Mục tiêu

So sánh hai cách xử lý đơn hàng trong cùng điều kiện:

| | Đồng bộ (đối chứng) | Bất đồng bộ (đề xuất) |
|---|---|---|
| Luồng | API ghi DynamoDB rồi gửi SES ngay trong request | API chỉ đưa đơn vào SQS; Lambda và Step Functions xử lý sau |
| Chống trùng | Không | Có (DynamoDB) |
| Thử lại, DLQ, cảnh báo | Không | Có |

Chỉ số: phản hồi API (trung bình, p95), đơn hoàn tất, đơn thất bại hoặc mất,
thời gian hoàn tất xử lý, số email dư khi gửi trùng, và thời gian có cảnh báo
khi lỗi.

>**Giới hạn của SES sandbox:** tối đa 1 email/giây. Email gửi đến
>`success@simulator.amazonses.com` (hộp thư giả lập của SES) không tính vào
>hạn mức 200 email/ngày nhưng vẫn bị giới hạn 1 email/giây. Vì vậy mức tải thử
>là 20 và 50 đơn. Đây cũng là một kết quả đáng ghi nhận: nút thắt nằm ở dịch
>vụ gửi email chứ không phải ở kiến trúc.

## Các phần thực hành

Mở từng mục dưới đây hoặc dùng menu bên trái. Thực hiện theo thứ tự; các bước sau có thể dùng tài nguyên và cấu hình từ bước trước.

- [5.7.1 Cập nhật template.yaml](step-01/)
- [5.7.2 Viết mã](step-02/)
- [5.7.3 Triển khai](step-03/)
- [5.7.4 Chuẩn bị công cụ đo](step-04/)
- [5.7.5 Thí nghiệm 1: độ trễ và tải](step-05/)
- [5.7.6 Thí nghiệm 2: gửi trùng đơn](step-06/)
- [5.7.7 Thí nghiệm 3: lỗi và cảnh báo](step-07/)
- [5.7.9 Kết luận](step-09/)
