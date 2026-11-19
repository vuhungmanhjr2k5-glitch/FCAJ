---
title: "Triển khai và kiểm tra hệ thống giám sát"
weight: 3
chapter: false
pre: " <b> 5.6.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Triển khai với tên tài nguyên lấy từ CloudFormation

Thực hiện trong thư mục `order-system` sau khi xong 5.5. Trước hết đọc Outputs và Resources của ứng dụng để lấy queue URL, ARN state machine, tên worker và ARN topic. Khối lệnh bên dưới tách tên queue ở cuối URL vì CloudWatch dùng tên làm dimension. Đặt tên riêng cho stack giám sát để sau này có thể xem sự kiện và dọn các alarm thuộc đúng stack.

```powershell
$region = 'ap-southeast-1'
$sourceStack = 'order-system'
$monitorStack = 'order-monitoring'
$stackJson = aws cloudformation describe-stacks --stack-name $sourceStack --region $region --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the application stack.' }
$outputs = @{}
foreach ($item in ($stackJson | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$resourcesJson = aws cloudformation list-stack-resources --stack-name $sourceStack --region $region --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read stack resources.' }
$physical = @{}
foreach ($item in ($resourcesJson | Out-String | ConvertFrom-Json).StackResourceSummaries) {
    $physical[$item.LogicalResourceId] = $item.PhysicalResourceId
}
foreach ($name in 'QueueUrl','DlqUrl','StateMachineArn') {
    if (-not $outputs[$name]) { throw "Missing stack output: $name" }
}
foreach ($name in 'ProcessOrderFn','OrderTopic') {
    if (-not $physical[$name]) { throw "Missing physical resource: $name" }
}
$parameters = @(
    "SourceStack=$sourceStack"
    "SourceQueueName=$($outputs.QueueUrl.Split('/')[-1])"
    "DlqName=$($outputs.DlqUrl.Split('/')[-1])"
    "WorkerName=$($physical.ProcessOrderFn)"
    "MachineArn=$($outputs.StateMachineArn)"
    "TopicArn=$($physical.OrderTopic)"
)
aws cloudformation deploy --template-file .\monitoring.template.json --stack-name $monitorStack --region $region --parameter-overrides @parameters --no-fail-on-empty-changeset --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Monitoring deployment failed.' }
$config = [ordered]@{
    Region = $region
    QueueUrl = $outputs.QueueUrl
    DlqUrl = $outputs.DlqUrl
    LogGroup = "/aws/lambda/$($physical.ProcessOrderFn)"
    AlarmNames = @("$monitorStack-sfn-failed", "$monitorStack-lambda-errors", "$monitorStack-dlq")
}
$configPath = Join-Path $PWD.Path 'workshop-monitoring-5-6.json'
[IO.File]::WriteAllText($configPath, ($config | ConvertTo-Json -Depth 4), [Text.UTF8Encoding]::new($false))
```

Khối lệnh lưu một tệp cấu hình cục bộ cho 5.6.4. Tệp chứa Region, URL hai queue, tên alarm và log group của worker. Khi chuyển tài khoản, Region hoặc thay stack ứng dụng, hãy chạy lại bước liên kết để tạo cấu hình mới. Không chép định danh của lần minh họa vào tài khoản khác.

## Kiểm tra trước khi gây lỗi

```powershell
$config = Get-Content -Raw -LiteralPath .\workshop-monitoring-5-6.json | ConvertFrom-Json
$names = @($config.AlarmNames)
aws cloudwatch describe-alarms --alarm-names @names --region $config.Region --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Enabled:ActionsEnabled}' --output table --no-cli-pager
```

Kết quả cần có ba alarm. Kiểm tra các dimension, `ActionsEnabled` và địa chỉ topic trong phần Details. Alarm mới có thể ở `INSUFFICIENT_DATA` trong lúc chờ đánh giá. Với lựa chọn `notBreaching`, một alarm có thể về `OK` dù luồng vừa triển khai chưa phát sinh dữ liệu. Xem cấu hình đúng rồi ghi trạng thái đầu làm mốc cho phép thử tiếp theo. Các hình trên trang ghi lại môi trường và thời điểm riêng của tác giả.

### Khi dashboard trống dù deploy đã xong

Mở một widget và xem Region, khoảng thời gian và dimension trong cấu hình. Nếu widget tham chiếu đúng queue nhưng chưa có traffic trong khoảng đã chọn, một biểu đồ trống chưa nói được gì về hệ thống gửi cảnh báo. Nếu tên queue khác tên trong Outputs của ứng dụng, chạy lại bước ánh xạ với đúng stack nguồn. Cuối cùng xem phần **Actions** của từng alarm. Có metric và có email được gửi là hai khâu cần kiểm tra riêng, nhất là khi người quản trị chưa xác nhận subscription.

## Lần triển khai đã ghi trong tài liệu

Alarm và dashboard CREATE_COMPLETE; stack order-system UPDATE_COMPLETE. Đây là minh chứng triển khai.

![Triển khai giám sát ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h1.png)

Ba alarm cũ ở OK và Actions enabled. Trạng thái này chưa chứng minh email đã được nhận.

[Trạng thái ba alarm ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h2.png)

Dashboard order-workshop hiển thị hàng đợi, tuổi thông điệp, Step Functions và Lambda.

[Dashboard ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h3.png)

Alarm order-sfn-failed ở ALARM, mốc cập nhật 31/10/2026 12:40:46 UTC. Không dùng mốc cũ này tính độ trễ của lần thử mới.

[Alarm Step Functions ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h4.png)
