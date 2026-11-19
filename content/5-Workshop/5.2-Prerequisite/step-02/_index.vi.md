---
title: "Cài đặt công cụ"
weight: 2
chapter: false
pre: " <b> 5.2.2 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Mở PowerShell và chạy:

```powershell
# Chọn Node.js 22 để khớp runtime của template.
winget install -e --id OpenJS.NodeJS.22
winget install -e --id Amazon.AWSCLI
winget install -e --id Amazon.SAM-CLI
```

Đóng và mở lại PowerShell, rồi kiểm tra:

```powershell
node -v
aws --version
sam --version
```

<!-- workshop-detail:v1:begin -->
## Vai trò của các công cụ

| Công cụ | Mục đích |
|---|---|
| AWS CLI v2 | Đọc cấu hình và thao tác tài nguyên AWS từ PowerShell |
| AWS SAM CLI | Build/deploy ứng dụng theo template SAM trong bài |
| Node.js và npm | Quản lý mã JavaScript và thư viện phụ thuộc |
| Git | Theo dõi thay đổi mã nguồn và tài liệu |
| Hugo | Xem và xuất bản tài liệu workshop; không xử lý đơn hàng AWS |

Sau khi cài đặt, đóng và mở lại terminal để cập nhật PATH. Chọn phiên bản Node.js phù hợp với mã dự án và runtime Lambda trong template. Việc máy đã cài một bản Node.js mới không tự thay đổi runtime của Lambda đã triển khai.

## Kiểm tra nhanh trên PowerShell

```powershell
aws --version
sam --version
node --version
npm.cmd --version
git --version
hugo version
```

Kết quả mong đợi là từng công cụ in thông tin phiên bản; AWS CLI hiển thị nhánh phiên bản 2. Nếu chỉ làm phần AWS thì Hugo phục vụ tài liệu có thể được kiểm tra riêng. Các lệnh này không tạo tài nguyên và không gửi email.

## Lỗi thường gặp

Nếu có thông báo command not found/not recognized, mở terminal mới và kiểm tra công cụ đã cài hoặc đã có trong PATH chưa. Chưa nên xử lý lỗi này bằng cách sửa template AWS. Khi chạy lệnh thực hành trên Windows, dùng đúng PowerShell, tránh trộn cú pháp Bash.

<!-- workshop-detail:v1:end -->

<!-- workshop-proof:v1:begin -->
## Minh chứng kiểm tra công cụ

{{< workshop-proof file="Version.png" alt="Kết quả kiểm tra phiên bản trên PowerShell" caption="PowerShell hiển thị AWS CLI 2.37.9, SAM CLI 1.166.2, Node.js v24.19.0 và Git 2.56.0.windows.1." >}}

Ảnh xác nhận bốn công cụ gọi được trên máy tại thời điểm kiểm tra. Ảnh không hiển thị phiên bản npm hoặc Hugo; cũng không chứng minh đã xác thực AWS hay xác định runtime Lambda đang triển khai.
<!-- workshop-proof:v1:end -->

## Tìm đúng chương trình khi máy có nhiều phiên bản

PowerShell tìm chương trình theo thứ tự trong PATH. Một máy có hai bản Node.js hoặc hai bản AWS CLI có thể trả kết quả khác giữa terminal cũ và terminal mới. Khối sau liệt kê đường dẫn thực tế. So sánh cột Source trước khi cài lại gói.

```powershell
Get-Command aws, sam, node, npm.cmd, git |
    Select-Object Name, Source, Version
```

Runtime trong template là `nodejs22.x`. Nếu SAM báo không tìm được bộ dựng Node.js phù hợp, dùng bản Node.js 22 trên máy dựng, mở một cửa sổ PowerShell mới rồi kiểm tra lại phiên bản. Trên Windows, khi lệnh npm bị chặn do shell chọn tệp `npm.ps1`, có thể dùng `npm.cmd` đi kèm Node.js. Các ví dụ ghi tệp và cài thư viện phía sau chạy được trong cùng cửa sổ đó.

## Phân biệt ba lần kiểm tra

Phiên bản công cụ cho biết máy chạy được chương trình. STS ở bước kế tiếp cho biết danh tính AWS đang dùng. Lệnh kiểm tra template ở mục 5.3 cho biết SAM có đọc được cấu hình của dự án. Nếu kiểm tra phiên bản thất bại, giải quyết ở máy tính trước. Nếu cả ba công cụ đã in phiên bản, chuyển đến cấu hình profile mà chưa cần tạo queue hoặc gửi email thử.
