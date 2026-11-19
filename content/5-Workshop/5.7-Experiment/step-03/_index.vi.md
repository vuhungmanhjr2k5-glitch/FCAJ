---
title: "Triển khai"
weight: 3
chapter: false
pre: " <b> 5.7.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Khóa dùng cho lần triển khai API

Khi tạo API lần đầu, khối dưới tạo khóa trong phiên PowerShell rồi đưa vào clipboard. Nếu muốn dùng lại khóa cũ, đặt biến `API_KEY` của phiên bằng giá trị đang dùng trước bước này.

```powershell
if ([string]::IsNullOrWhiteSpace($env:API_KEY)) {
    $env:API_KEY = [guid]::NewGuid().ToString('N')
}
Set-Clipboard -Value $env:API_KEY
```

Ở màn hình guided deploy, giữ `order-system` và hai email đã dùng. Dán giá trị từ clipboard khi được hỏi `ApiKey`. Hai API được bảo vệ bằng khóa ứng dụng. Giữ khóa và tệp `samconfig.toml` ở máy thực hành.

```powershell
sam validate --region ap-southeast-1
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed.' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed.' }
Set-Clipboard -Value $env:API_KEY
sam deploy --guided --region ap-southeast-1 --no-fail-on-empty-changeset
if ($LASTEXITCODE -ne 0) { throw 'Inspect the CloudFormation deployment events.' }
```

### Lấy lại Outputs sau khi triển khai thành công

```powershell
$stackData = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read order-system.' }
$outputs = @{}
foreach ($item in ($stackData | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$api = $outputs['ApiUrl']
if (-not $api) { throw 'Missing output: ApiUrl' }
$table = $outputs['TableName']
if (-not $table) { throw 'Missing output: TableName' }
$sm = $outputs['StateMachineArn']
if (-not $sm) { throw 'Missing output: StateMachineArn' }
$env:AWS_REGION = 'ap-southeast-1'
$api = $api.TrimEnd('/')
```

## Tách cấu hình triển khai khỏi biến dùng để đo

`ApiKey` là tham số CloudFormation mà hai hàm so với header `x-api-key`. `$env:API_KEY` là biến trong cửa sổ PowerShell để công cụ đo xây dựng đúng header đó. Hai nơi phải có cùng giá trị. Cờ `NoEcho` che giá trị ở một số màn hình CloudFormation; bản lưu guided deploy hoặc lịch sử máy cá nhân vẫn cần được lưu giữ cẩn thận. Nếu triển khai lại không đổi khóa, dùng giá trị đang cấu hình để tránh làm hỏng bộ công cụ đo của buổi hiện tại.

Địa chỉ trong `ApiUrl` là URL gốc. Khi gọi thử, nối một lần `/async` hoặc `/sync` ở cuối. Với mỗi lần chạy, lấy URL từ Outputs của cùng stack đã cung cấp `TableName` và `StateMachineArn`. Tên có hậu tố `evidence` trong ảnh minh họa có thể khác tài nguyên bạn đang dùng.

## Thử một yêu cầu trước khi đo hàng loạt

```powershell
$smokeId = 'smoke-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$body = @{ orderId = $smokeId; customerEmail = 'success@simulator.amazonses.com' } | ConvertTo-Json -Compress
$response = Invoke-WebRequest -UseBasicParsing -Method Post -Uri "$api/async" -Headers @{'x-api-key' = $env:API_KEY} -ContentType 'application/json' -Body $body -ErrorAction Stop
$response | Select-Object StatusCode, Content
```

Đọc `StatusCode` và body trả về. Đường bất đồng bộ cần trả 202 và mã đơn bạn vừa gửi. Ghi mã đó và tìm kết quả cuối trong DynamoDB hoặc Step Functions trước khi chạy bộ đo. Nếu nhận 403, kiểm tra giá trị khóa ở phiên hiện tại. Với 404, xem URL gốc, đường dẫn và method POST. Với 500, dùng log của đúng hàm để xác định lỗi hạ tầng hoặc quyền. Không đưa lượt thử đầu này vào nhóm đơn dùng tính P95.

<!-- evidence-deployment:start -->

**Triển khai bổ sung:** stack order-api-evidence dùng lại tài nguyên order-orchestration-evidence. Ảnh và số liệu bên dưới thuộc đợt 13/11/2026.

![Triển khai hai API so sánh](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-api-deploy-success.png)

<!-- evidence-deployment:end -->
