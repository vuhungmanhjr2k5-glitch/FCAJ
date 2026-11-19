---
title: "Kết nối máy với tài khoản AWS"
weight: 3
chapter: false
pre: " <b> 5.2.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

1. Vào IAM → Users → chọn user → *Security credentials* → *Create access key*
   (use case: Command Line Interface).
2. Cấu hình profile:

```powershell
aws configure --profile order-workshop
$env:AWS_PROFILE = "order-workshop"
aws sts get-caller-identity
```

Nhập Access Key, Secret Key, region ap-southeast-1, output json.
Kết quả của lệnh cuối phải hiển thị thông tin tài khoản của bạn.

![Kiểm tra danh tính](/images/5-Workshop/5.2-Prerequisite/002.png)


Không đưa Access Key lên GitHub. Hãy xóa hoặc vô hiệu hóa key sau khi hoàn thành workshop.

<!-- workshop-detail:v1:begin -->
## Xác định danh tính và Region

AWS Console và AWS CLI có thể đang dùng hai danh tính khác nhau. Vì vậy, trước khi triển khai cần kiểm tra tài khoản mà lệnh CLI thực sự sử dụng. Nếu tổ chức cấp phiên đăng nhập qua IAM Identity Center, sử dụng profile được tổ chức hướng dẫn; không cần tạo thêm access key chỉ để theo một ví dụ.

Với profile order-workshop đã được cấu hình, có thể kiểm tra:

```powershell
aws sts get-caller-identity --profile order-workshop --region ap-southeast-1 --query '{Account:Account,Arn:Arn}' --output table --no-cli-pager
aws configure get region --profile order-workshop
```

Lệnh đầu phải trả về đúng Account và Arn mong muốn. Lệnh thứ hai đọc Region lưu trong profile; kết quả trống nghĩa là profile chưa lưu giá trị này, không phải xác nhận Region trên trình duyệt. Trong bài, tham số --region ap-southeast-1 giúp xác định vùng rõ ràng.

## Tránh triển khai nhầm

Dùng cùng profile cho các bước tiếp theo. Nếu đã chọn profile bằng biến AWS_PROFILE thì kiểm tra lại sau khi mở một cửa sổ PowerShell mới, vì biến của phiên trước có thể không còn. Khi gặp ExpiredToken, làm mới phiên đăng nhập; khi STS thành công nhưng thao tác dịch vụ bị AccessDenied, kiểm tra quyền riêng của dịch vụ.

Không đưa access key, secret key hoặc session token vào mã nguồn và ảnh báo cáo. Nếu máy đã kết nối thành công thì kiểm tra lại là đủ, không cần tạo thêm khóa.

<!-- workshop-detail:v1:end -->

## Mở lại PowerShell vào ngày hôm sau

Tệp cấu hình profile được lưu trên máy, còn biến môi trường gán trong cửa sổ PowerShell chỉ có hiệu lực trong phiên đó và các tiến trình con. Mỗi khi mở shell mới để làm các mục 5.3–5.7, chạy khối sau. Nếu lớp học cấp profile với tên khác, thay đúng tên ở dòng đầu.

```powershell
$env:AWS_PROFILE = 'order-workshop'
$env:AWS_REGION = 'ap-southeast-1'
$env:AWS_DEFAULT_REGION = $env:AWS_REGION
aws sts get-caller-identity --output table --no-cli-pager
```

Đọc `Account` trong kết quả và so với tài khoản thực hành. ARN dạng assumed-role có tên phiên ở phần cuối. Nếu đăng nhập qua IAM Identity Center và phiên đã hết hạn, đăng nhập lại bằng đúng profile rồi chạy lại khối kiểm tra. Việc chọn Singapore trên trình duyệt chỉ đổi giao diện Console. Biến Region của PowerShell giúp cả AWS CLI và các công cụ Node.js ở phần đo tải làm việc tại Singapore.

## Khi tài khoản đúng nhưng không thấy stack

Hãy kiểm tra ba chi tiết theo thứ tự: tên profile trong shell vừa dùng, Region gắn với lệnh và tên stack trong buổi thực hành. Một tên stack có thể xuất hiện ở nhiều Region. Việc lệnh STS chạy được nhưng danh sách SQS trống là dấu hiệu cần xem lại phạm vi tài nguyên. Chỉ bắt đầu triển khai khi dòng Account và Region đều khớp với ghi chép của buổi học.
