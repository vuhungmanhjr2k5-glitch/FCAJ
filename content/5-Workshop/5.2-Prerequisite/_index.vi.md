---
title: "Chuẩn bị (IAM, region, công cụ)"
date: "2026-10-03"
weight: 2
chapter: false
pre: " <b> 5.2 </b> "
---

<!-- workshop-split:v1 -->

## Yêu cầu chuẩn bị

- Một tài khoản AWS dùng cho thử nghiệm (không dùng tài khoản production)
- Một địa chỉ email thật để nhận thư kiểm thử
- Máy tính Windows có PowerShell, đã cài Git
- Region sử dụng trong toàn bộ workshop: **Asia Pacific (Singapore) ap-southeast-1**

<!-- workshop-detail:v1:begin -->
## Mục đích của bước chuẩn bị

Phần này giúp các bước triển khai sau dùng đúng danh tính, đúng Region và đúng môi trường công cụ. Chuẩn bị tốt giúp phân biệt lỗi của thiết kế với lỗi do thiếu quyền, chưa xác minh email hoặc cấu hình máy chưa hoàn tất.

## Những yếu tố cần thống nhất

| Yếu tố | Quy ước của bài thực hành |
|---|---|
| Region | Singapore, mã ap-southeast-1 |
| Tài khoản | Tài khoản thực nghiệm được phép sử dụng |
| Profile CLI | Dùng nhất quán profile đã cấu hình, ví dụ order-workshop |
| Email | Người gửi và người nhận đáp ứng điều kiện SES của Region |
| Thư mục | Phân biệt dự án AWS với dự án tài liệu Hugo |

Region được chọn trên trình duyệt không tự đổi Region trong PowerShell. Khi kiểm tra tài nguyên, nên truyền rõ --region ap-southeast-1. IAM là dịch vụ toàn cục, còn SQS, Lambda, DynamoDB, Step Functions và trạng thái Sandbox SES liên quan đến Region đang sử dụng.

## Điều kiện chuyển sang mục 5.3

Các lệnh kiểm tra công cụ trả kết quả; STS hiển thị đúng tài khoản; email thử đã sẵn sàng; người triển khai có quyền cần thiết; có nơi lưu mã nguồn và minh chứng. Các ảnh chuẩn bị có thể được chụp cùng một lần kiểm tra, không cần tạo tài nguyên mới chỉ để minh họa.
<!-- workshop-detail:v1:end -->

## Các phần thực hành

Mở từng mục dưới đây hoặc dùng menu bên trái. Thực hiện theo thứ tự; các bước sau có thể dùng tài nguyên và cấu hình từ bước trước.

- [5.2.1 Quyền IAM](step-01/)
- [5.2.2 Cài đặt công cụ](step-02/)
- [5.2.3 Kết nối máy với tài khoản AWS](step-03/)
- [5.2.4 Xác minh email trong Amazon SES](step-04/)
- [5.2.5 Đặt cảnh báo chi phí](step-05/)
- [5.2.6 Tạo thư mục dự án](step-06/)
