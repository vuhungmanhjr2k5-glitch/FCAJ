---
title: "Cách thực nghiệm"
weight: 4
chapter: false
pre: " <b> 5.6.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Cần AWS CLI v2, stack nguồn ổn định, SES sender đã xác minh và SNS email đã xác nhận. Dùng cấu hình `workshop-monitoring-5-6.json` đã tạo khi triển khai. Script dưới đây lưu trạng thái alarm/DLQ trước khi gửi, rồi gửi ba thông điệp với mã mới cho mỗi lần chạy.

| Trường hợp | Dữ liệu | Kỳ vọng cần kiểm tra |
|---|---|---|
| Thành công | Email SES simulator hợp lệ | Execution SUCCEEDED và đơn DONE |
| Lỗi gửi email | not-an-email | Lỗi/thử lại tại Step Functions, execution FAILED, alarm sfn-failed |
| JSON sai | Chuỗi not-json-mon56-... | Lambda phát sinh lỗi đọc JSON, SQS retry, thông điệp vào DLQ |

Với code mục 5.5, Lambda trả về sau khi khởi động workflow. SES lỗi về sau không nhất thiết tăng Lambda Errors. Phép thử JSON sai kiểm tra riêng Lambda và DLQ. MessageId từ SQS chỉ chứng minh tiếp nhận, chưa chứng minh xử lý hoàn tất.

```powershell
& {
  param([string]$ConfigPath = (Join-Path $PWD.Path 'workshop-monitoring-5-6.json'))
  $ErrorActionPreference = 'Stop'
  function Read-Aws([string[]]$Arguments) {
    $raw = & aws @Arguments --region $c.Region --output json --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw ('AWS CLI failed: '+($Arguments -join ' ')) }
    return (($raw -join [Environment]::NewLine) | ConvertFrom-Json)
  }
  try {
    $c = Get-Content -LiteralPath $ConfigPath -Raw | ConvertFrom-Json
    $id = 'mon56-'+(Get-Date -Format 'yyyyMMdd-HHmmss')+'-'+[guid]::NewGuid().ToString('N').Substring(0,6)
    $dir = Join-Path (Split-Path -Parent $ConfigPath) $id
    $null = New-Item -ItemType Directory -Path $dir
    $utf8 = New-Object System.Text.UTF8Encoding($false)
    $before = Read-Aws -Arguments (@('cloudwatch','describe-alarms','--alarm-names') + @($c.AlarmNames))
    [IO.File]::WriteAllText((Join-Path $dir 'alarms-before.json'),($before | ConvertTo-Json -Depth 30),$utf8)
    $queueBefore = Read-Aws -Arguments @('sqs','get-queue-attributes','--queue-url',$c.DlqUrl,'--attribute-names','ApproximateNumberOfMessages','ApproximateNumberOfMessagesNotVisible')
    [IO.File]::WriteAllText((Join-Path $dir 'dlq-before.json'),($queueBefore | ConvertTo-Json -Depth 10),$utf8)
    Write-Host '=== ALARMS BEFORE TEST ===' -ForegroundColor Cyan
    $before.MetricAlarms | Select-Object AlarmName,StateValue | Format-Table -AutoSize
    $good = @{orderId=($id+'-ok');customerEmail='success@simulator.amazonses.com';acceptedAt=[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()}
    $bad = @{orderId=($id+'-ses-fail');customerEmail='not-an-email';acceptedAt=[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()}
    $cases = @(
      @{Name='success';Body=($good | ConvertTo-Json -Compress)},
      @{Name='ses-failure';Body=($bad | ConvertTo-Json -Compress)},
      @{Name='malformed-json';Body=('not-json-'+$id)}
    )
    $results = @()
    foreach ($case in $cases) {
      $file = Join-Path $dir ($case.Name+'.txt')
      [IO.File]::WriteAllText($file,$case.Body,$utf8)
      $sent = Read-Aws -Arguments @('sqs','send-message','--queue-url',$c.QueueUrl,'--message-body',('file://'+$file))
      $results += [pscustomobject]@{Case=$case.Name;MessageId=$sent.MessageId;SentAtUtc=[DateTimeOffset]::UtcNow.ToString('o')}
      [IO.File]::WriteAllText((Join-Path $dir 'sent.json'),($results | ConvertTo-Json -Depth 10),$utf8)
    }
    $run = @{RunId=$id;GoodOrderId=$good.orderId;FailedOrderId=$bad.orderId;MalformedBody=('not-json-'+$id);EvidenceDirectory=$dir}
    [IO.File]::WriteAllText((Join-Path $dir 'run.json'),($run | ConvertTo-Json -Depth 10),$utf8)
    Write-Host '=== TEST MESSAGES ACCEPTED BY SQS ===' -ForegroundColor Green
    $results | Format-Table -AutoSize
    $run | Format-List
    Write-Host 'Da gui 3 tin; chua ket luan xu ly thanh cong. Theo doi Step Functions, Lambda, DLQ va CloudWatch trong 5-10 phut.'

    Write-Host 'DLQ co san tin cu thi ALARM/email cu khong chung minh do tin vua gui.'
  } catch {
    Write-Host ('LOI: '+$_.Exception.Message) -ForegroundColor Red
    throw
  }
}
```

Theo dõi ban đầu trong 5–10 phút, cho phép thêm thời gian cho retry và cập nhật metric. Tìm execution theo orderId; đối chiếu Lambda Logs, tên queue và body trong DLQ; đọc alarm Details/History và email có đúng tên alarm. Không dùng set-alarm-state để làm minh chứng sự cố thật.

## Lập dự đoán cho ba tín hiệu

Đơn hợp lệ dùng simulator sẽ tạo một workflow có thể đi đến trạng thái thành công. Tin có địa chỉ sai vẫn là JSON hợp lệ nên Lambda có thể khởi động workflow thành công trước khi SES từ chối. Tin thứ ba dừng ngay ở parse. Do đó cả ba được SQS chấp nhận, nhưng chỉ hai tin đầu có thể có một execution mang mã đơn. Nếu bạn tìm history cho chuỗi văn bản hỏng và không thấy, hãy chuyển điểm tra cứu sang log Lambda.

Khi chạy block một lần, các body và kết quả gửi được lưu chung trong một thư mục có mã riêng. Mở `alarms-before.json` để biết alarm nào đã ở `ALARM`. Mở `dlq-before.json` xem số tin visible và in-flight trước thử. Nếu đã có tồn đọng, giữ nguyên thông tin nền đó. Không cần xóa tin cũ để tạo một biểu đồ dễ đọc.

## Xây dựng dòng thời gian sau khi gửi

| Mốc | Nguồn | Cách dùng |
|---|---|---|
| Gửi thành công | `sent.json` và MessageId | Xác định thời điểm phía máy khách ghi nhận SQS chấp nhận |
| Lỗi nghiệp vụ | History của workflow hoặc log parse | Gắn lỗi đầu với đúng trường hợp |
| Đổi trạng thái alarm | CloudWatch alarm history | Xác định alarm nào thực sự chuyển trạng thái |
| Nhận thư | Email SNS có cùng tên alarm | Ghi mốc nhận nếu muốn đánh giá kênh thông báo |

Nếu có khoảng trống ở một mốc, giữ các mốc đã có và ghi “chưa quan sát”. Lặp script sẽ sinh bộ mã mới cùng ba tin khác. Đọc kết quả lượt hiện tại trước khi tạo lượt tiếp. Điều này đặc biệt quan trọng khi alarm vẫn đang ở `ALARM` và sẽ không phát sinh thêm lần chuyển trạng thái tương ứng với mỗi tin.

### Mở lại kết quả mà không gửi thêm tin

Lấy đường dẫn `EvidenceDirectory` ở cuối lần chạy và nhập vào khi được hỏi. Khối sau chỉ đọc dữ liệu cục bộ. So sánh ba dòng trong `sent.json` với các mã mà `run.json` ghi lại. Một lần chạy dừng ngay sau lần gửi thứ nhất có thể chỉ có một biên nhận; chỉ các trường hợp có bằng chứng đã gửi mới được đưa vào đối chiếu.

```powershell
$dir = Read-Host 'EvidenceDirectory'
Get-Content -LiteralPath (Join-Path $dir 'sent.json') -Raw | ConvertFrom-Json | Format-Table Case, MessageId, SentAtUtc
$runFile = Join-Path $dir 'run.json'
if (Test-Path -LiteralPath $runFile) {
    Get-Content -LiteralPath $runFile -Raw | ConvertFrom-Json | Format-List
}
```

Trường `SentAtUtc` được ghi sau khi lệnh gửi nhận phản hồi, nên đây là mốc quan sát phía máy khách. Khi lập timeline chi tiết, không gán giá trị này cho thời điểm Lambda bắt đầu chạy. Nếu thư mục không có `run.json` do lần chạy bị gián đoạn, giữ `sent.json` và body đã ghi làm dấu vết. Trước khi thử lại, tìm các mã cũ để biết một số công việc có đang chạy tiếp trên AWS hay không.

### Đọc log của worker cho lượt vừa gửi

Chạy tại thư mục ứng dụng. Giữ mốc giờ của lượt thử bên cạnh cửa sổ log để phân biệt các invocation của các lượt khác nhau.

```powershell
$config = Get-Content -Raw -LiteralPath .\workshop-monitoring-5-6.json | ConvertFrom-Json
aws logs tail $config.LogGroup --since 15m --region $config.Region --format short --no-cli-pager
```
