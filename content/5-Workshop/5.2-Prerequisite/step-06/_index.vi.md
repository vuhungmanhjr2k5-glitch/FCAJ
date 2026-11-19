---
title: "Tạo thư mục dự án"
weight: 6
chapter: false
pre: " <b> 5.2.6 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

```powershell
$appDir = Join-Path $env:USERPROFILE 'Downloads\order-system'
New-Item -ItemType Directory -Path $appDir -Force -ErrorAction Stop | Out-Null
Set-Location -LiteralPath $appDir -ErrorAction Stop
New-Item -ItemType Directory -Path .\src -Force -ErrorAction Stop | Out-Null
```

Ở các mục tiếp theo, ta sẽ thêm `template.yaml` và mã nguồn Lambda vào thư mục này.

<!-- workshop-detail:v1:begin -->
## Tổ chức mã nguồn và tài liệu

Nên phân biệt ba khu vực: dự án AWS của workshop, bộ website Mộc và dự án Hugo chứa tài liệu. Template và mã Lambda nằm trong dự án AWS; các file _index.vi.md và hình minh chứng nằm trong Hugo. Việc sửa bài viết không yêu cầu triển khai lại tài nguyên AWS.

| Đường dẫn trong `order-system` | Nội dung |
|---|---|
| `template.yaml` | Tài nguyên do SAM triển khai |
| `src/index.mjs` | Handler của Lambda đọc đơn |
| `src/package.json` | Thư viện của hàm |
| `samconfig.toml` | Cấu hình đã lưu sau guided deploy |

Đây là cấu trúc định hướng. Các file được tạo dần trong những mục tiếp theo, không phải tất cả đều phải xuất hiện ngay sau bước tạo thư mục.

## Kiểm tra thư mục đang sử dụng

```powershell
Get-Location
Get-ChildItem
```

Trước mỗi bước build/deploy, kiểm tra đang đứng trong thư mục chứa template cần dùng. Nếu thư mục hoặc mã nguồn đã có từ lần thực hành trước, sử dụng lại và kiểm tra nội dung; không tạo file rỗng đè lên template hay mã Lambda đang hoạt động.

<!-- workshop-detail:v1:end -->

## Mở đúng thư mục ở lần thực hành kế tiếp

Bài này dùng hai thư mục ngang hàng trong Downloads. `Workshop-AWS-ZIP` là dự án Hugo đang đọc. `order-system` là nơi chứa template, mã Lambda và công cụ đo sẽ tạo trong các bước sau. Từ một PowerShell mới, khối dưới đưa bạn vào đúng thư mục ứng dụng và hiển thị các tệp cấp đầu.

```powershell
$appDir = Join-Path $env:USERPROFILE 'Downloads\order-system'
Set-Location -LiteralPath $appDir -ErrorAction Stop
Get-Location
Get-ChildItem -Name
```

Ở cuối mục 5.2, việc thư mục `src` còn trống là bình thường. Sau mục 5.3.1 phải thấy `template.yaml` cùng cấp với `src`, sau mục 5.4.2 mới cần có `src/index.mjs`. Sau lần deploy có lưu cấu hình, `samconfig.toml` nằm cạnh template. Nếu chỉ thấy các thư mục `content`, `themes` hoặc `static`, bạn đang ở dự án tài liệu.

## Khi quay lại một dự án đang làm dở

Trước khi chạy khối ghi tệp của một chương sau, mở đầu tệp hiện có và xem nó thuộc phiên bản nào của bài. Ví dụ template có tham số `SenderEmail` đã tới phần điều phối. Lưu thay đổi riêng bằng Git hoặc bản sao trước khi dùng khối thay toàn bộ tệp. Để chẩn đoán lỗi build, ghi đường dẫn làm việc và tên tệp vừa sửa. Hai thông tin đó có giá trị hơn ảnh chụp cả màn hình trình soạn thảo.
