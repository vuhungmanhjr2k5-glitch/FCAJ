---
title: "Xóa stack quản lý của SAM (tùy chọn)"
weight: 3
chapter: false
pre: " <b> 5.8.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

SAM tạo thêm stack `aws-sam-cli-managed-default` kèm một S3 bucket chứa mã
triển khai. Chi phí rất nhỏ nhưng bạn có thể dọn: vào **S3**, mở bucket có tên
bắt đầu bằng `aws-sam-cli-managed-default`, chọn **Empty**, sau đó vào
**CloudFormation** xóa stack `aws-sam-cli-managed-default`.
