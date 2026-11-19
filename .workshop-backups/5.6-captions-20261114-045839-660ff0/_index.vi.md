---
title: "Giám sát"
weight: 6
chapter: false
pre: " <b> 5.6 </b> "
---

## Mục tiêu và kiến trúc giám sát

Theo dõi SQS, Lambda và Step Functions bằng CloudWatch metric, ba alarm, một dashboard và email SNS tại **ap-southeast-1**. Stack giám sát bổ sung là **order-monitoring-evidence**, dùng tài nguyên ứng dụng của **order-orchestration-evidence**.

H1–H4 thuộc lần triển khai cũ **order-system**. H5–H11 thuộc phần giám sát bổ sung. Hai nhóm được trình bày riêng để tránh ghép thành cùng một chuỗi trước/sau.

## 1. Cấu hình metric và alarm

| Alarm | Namespace / metric | Dimension | Statistic |
|---|---|---|---|
| order-monitoring-evidence-sfn-failed | AWS/States / ExecutionsFailed | StateMachineArn | Sum |
| order-monitoring-evidence-lambda-errors | AWS/Lambda / Errors | FunctionName | Sum |
| order-monitoring-evidence-dlq | AWS/SQS / ApproximateNumberOfMessagesVisible | QueueName của DLQ | Maximum |

Cả ba dùng ngưỡng **> 0**, Period **60 giây**, EvaluationPeriods **1**, DatapointsToAlarm **1**, TreatMissingData **notBreaching**. AlarmActions và OKActions trỏ tới SNS topic của ứng dụng.

Sum cộng số lỗi trong kỳ; Maximum phát hiện số thông điệp nhìn thấy trong DLQ. Metric SQS là xấp xỉ. Period 60 giây không bảo đảm email tới trong 60 giây. CloudWatch thường gửi thông báo khi chuyển trạng thái, không gửi lại cho mỗi lỗi trong lúc alarm vẫn ALARM.

### Template triển khai

Tệp dưới đây được sao chép từ JSON tạo trên máy thực hành. Các định danh tài nguyên gắn với môi trường này; cần điều chỉnh khi dùng môi trường khác.

[monitoring.template.json](/downloads/5.6-Monitoring/evidence-20261114-043035-985e31/monitoring.template.json)

```json
{
    "Outputs":  {
                    "DashboardName":  {
                                          "Value":  "order-monitoring-evidence-dashboard"
                                      },
                    "SourceStack":  {
                                        "Value":  "order-orchestration-evidence"
                                    }
                },
    "Description":  "Workshop 5.6 monitoring for order-orchestration-evidence",
    "AWSTemplateFormatVersion":  "2010-09-09",
    "Resources":  {
                      "OrderDashboard":  {
                                             "Properties":  {
                                                                "DashboardName":  "order-monitoring-evidence-dashboard",
                                                                "DashboardBody":  "{\"widgets\":[{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Maximum\",\"metrics\":[[\"AWS/SQS\",\"ApproximateNumberOfMessagesVisible\",\"QueueName\",\"order-orchestration-evidence-OrdersQueue-Q4yad2S16mVq\",{\"label\":\"Source queue\"}],[\"AWS/SQS\",\"ApproximateNumberOfMessagesVisible\",\"QueueName\",\"order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU\",{\"label\":\"DLQ\"}]],\"period\":60,\"title\":\"SQS: visible messages\",\"view\":\"timeSeries\"},\"y\":0,\"type\":\"metric\",\"x\":0,\"height\":6},{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Maximum\",\"metrics\":[[\"AWS/SQS\",\"ApproximateAgeOfOldestMessage\",\"QueueName\",\"order-orchestration-evidence-OrdersQueue-Q4yad2S16mVq\"]],\"period\":60,\"title\":\"SQS: oldest source message (seconds)\",\"view\":\"timeSeries\"},\"y\":0,\"type\":\"metric\",\"x\":12,\"height\":6},{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Sum\",\"metrics\":[[\"AWS/States\",\"ExecutionsStarted\",\"StateMachineArn\",\"arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly\"],[\"AWS/States\",\"ExecutionsSucceeded\",\"StateMachineArn\",\"arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly\"],[\"AWS/States\",\"ExecutionsFailed\",\"StateMachineArn\",\"arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly\"]],\"period\":60,\"title\":\"Step Functions: executions\",\"view\":\"timeSeries\"},\"y\":6,\"type\":\"metric\",\"x\":0,\"height\":6},{\"width\":12,\"properties\":{\"legend\":{\"position\":\"bottom\"},\"region\":\"ap-southeast-1\",\"stat\":\"Sum\",\"metrics\":[[\"AWS/Lambda\",\"Invocations\",\"FunctionName\",\"order-orchestration-evidence-ProcessOrderFn-A3d416bkFPjD\"],[\"AWS/Lambda\",\"Errors\",\"FunctionName\",\"order-orchestration-evidence-ProcessOrderFn-A3d416bkFPjD\"]],\"period\":60,\"title\":\"Lambda: invocations and errors\",\"view\":\"timeSeries\"},\"y\":6,\"type\":\"metric\",\"x\":12,\"height\":6}]}"
                                                            },
                                             "Type":  "AWS::CloudWatch::Dashboard"
                                         },
                      "SfnFailedAlarm":  {
                                             "Properties":  {
                                                                "AlarmName":  "order-monitoring-evidence-sfn-failed",
                                                                "ComparisonOperator":  "GreaterThanThreshold",
                                                                "OKActions":  [
                                                                                  "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                              ],
                                                                "AlarmActions":  [
                                                                                     "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                                 ],
                                                                "Dimensions":  [
                                                                                   {
                                                                                       "Value":  "arn:aws:states:ap-southeast-1:397303793104:stateMachine:OrderStateMachine-vjnt41lqqHly",
                                                                                       "Name":  "StateMachineArn"
                                                                                   }
                                                                               ],
                                                                "EvaluationPeriods":  1,
                                                                "Namespace":  "AWS/States",
                                                                "AlarmDescription":  "Workshop 5.6; source stack: order-orchestration-evidence",
                                                                "TreatMissingData":  "notBreaching",
                                                                "MetricName":  "ExecutionsFailed",
                                                                "Statistic":  "Sum",
                                                                "Threshold":  0,
                                                                "DatapointsToAlarm":  1,
                                                                "Period":  60
                                                            },
                                             "Type":  "AWS::CloudWatch::Alarm"
                                         },
                      "DlqAlarm":  {
                                       "Properties":  {
                                                          "AlarmName":  "order-monitoring-evidence-dlq",
                                                          "ComparisonOperator":  "GreaterThanThreshold",
                                                          "OKActions":  [
                                                                            "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                        ],
                                                          "AlarmActions":  [
                                                                               "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                           ],
                                                          "Dimensions":  [
                                                                             {
                                                                                 "Value":  "order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU",
                                                                                 "Name":  "QueueName"
                                                                             }
                                                                         ],
                                                          "EvaluationPeriods":  1,
                                                          "Namespace":  "AWS/SQS",
                                                          "AlarmDescription":  "Workshop 5.6; source stack: order-orchestration-evidence",
                                                          "TreatMissingData":  "notBreaching",
                                                          "MetricName":  "ApproximateNumberOfMessagesVisible",
                                                          "Statistic":  "Maximum",
                                                          "Threshold":  0,
                                                          "DatapointsToAlarm":  1,
                                                          "Period":  60
                                                      },
                                       "Type":  "AWS::CloudWatch::Alarm"
                                   },
                      "LambdaErrorAlarm":  {
                                               "Properties":  {
                                                                  "AlarmName":  "order-monitoring-evidence-lambda-errors",
                                                                  "ComparisonOperator":  "GreaterThanThreshold",
                                                                  "OKActions":  [
                                                                                    "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                                ],
                                                                  "AlarmActions":  [
                                                                                       "arn:aws:sns:ap-southeast-1:397303793104:order-orchestration-evidence-OrderTopic-1gXMLibGVdC0"
                                                                                   ],
                                                                  "Dimensions":  [
                                                                                     {
                                                                                         "Value":  "order-orchestration-evidence-ProcessOrderFn-A3d416bkFPjD",
                                                                                         "Name":  "FunctionName"
                                                                                     }
                                                                                 ],
                                                                  "EvaluationPeriods":  1,
                                                                  "Namespace":  "AWS/Lambda",
                                                                  "AlarmDescription":  "Workshop 5.6; source stack: order-orchestration-evidence",
                                                                  "TreatMissingData":  "notBreaching",
                                                                  "MetricName":  "Errors",
                                                                  "Statistic":  "Sum",
                                                                  "Threshold":  0,
                                                                  "DatapointsToAlarm":  1,
                                                                  "Period":  60
                                                              },
                                               "Type":  "AWS::CloudWatch::Alarm"
                                           }
                  }
}
```

Triển khai lại cùng stack bằng tệp trong Downloads:

```powershell
$template = Join-Path $env:USERPROFILE 'Downloads\workshop-monitoring-5-6.template.json'
aws cloudformation deploy --template-file $template --stack-name order-monitoring-evidence --region ap-southeast-1 --no-fail-on-empty-changeset --no-cli-pager
```

## 2. Bốn ảnh triển khai ban đầu

**H1.** Alarm và dashboard CREATE_COMPLETE; stack order-system UPDATE_COMPLETE. Đây là minh chứng triển khai.

![H1 - Triển khai giám sát ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h1.png)

**H2.** Ba alarm cũ ở OK và Actions enabled. Trạng thái này chưa chứng minh email đã được nhận.

![H2 - Trạng thái ba alarm ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h2.png)

**H3.** Dashboard order-workshop hiển thị hàng đợi, tuổi thông điệp, Step Functions và Lambda.

![H3 - Dashboard ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h3.png)

**H4.** Alarm order-sfn-failed ở ALARM, mốc cập nhật 31/10/2026 12:40:46 UTC. Không dùng mốc cũ này tính độ trễ của lần thử mới.

![H4 - Alarm Step Functions ban đầu](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h4.png)

## 3. Cách thực nghiệm

Cần AWS CLI v2, stack nguồn ổn định, SES sender đã xác minh và SNS email đã xác nhận. Dùng cấu hình Downloads/workshop-monitoring-5-6.json đã tạo khi triển khai. Script dưới đây lưu trạng thái alarm/DLQ trước khi gửi, rồi gửi ba thông điệp với mã mới cho mỗi lần chạy.

| Trường hợp | Dữ liệu | Kỳ vọng cần kiểm tra |
|---|---|---|
| Thành công | Email SES simulator hợp lệ | Execution SUCCEEDED và đơn DONE |
| Lỗi gửi email | not-an-email | Lỗi/thử lại tại Step Functions, execution FAILED, alarm sfn-failed |
| JSON sai | Chuỗi not-json-mon56-... | Lambda phát sinh lỗi đọc JSON, SQS retry, thông điệp vào DLQ |

Với code mục 5.5, Lambda trả về sau khi khởi động workflow. SES lỗi về sau không nhất thiết tăng Lambda Errors. Phép thử JSON sai kiểm tra riêng Lambda và DLQ. MessageId từ SQS chỉ chứng minh tiếp nhận, chưa chứng minh xử lý hoàn tất.

```powershell
& {
  param([string]$ConfigPath = (Join-Path $env:USERPROFILE 'Downloads\workshop-monitoring-5-6.json'))
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
    Write-Host ('Lambda logs: aws logs tail "'+$c.LogGroup+'" --since 15m --region '+$c.Region+' --format short --no-cli-pager')
    Write-Host 'DLQ co san tin cu thi ALARM/email cu khong chung minh do tin vua gui.'
  } catch {
    Write-Host ('LOI: '+$_.Exception.Message) -ForegroundColor Red
    Write-Host $_.InvocationInfo.PositionMessage -ForegroundColor Yellow
  }
}
```

Theo dõi ban đầu trong 5–10 phút, cho phép thêm thời gian cho retry và cập nhật metric. Tìm execution theo orderId; đối chiếu Lambda Logs, tên queue và body trong DLQ; đọc alarm Details/History và email có đúng tên alarm. Không dùng set-alarm-state để làm minh chứng sự cố thật.

## 4. Kết quả quan sát H5–H11

Mã có trong thông điệp thử: **mon56-20261114-040056-316515**. Dashboard/alarm dùng UTC; hộp thư và thời gian SQS trong ảnh dùng UTC+07:00. Ví dụ 13/11/2026 21:03:44 UTC tương ứng 14/11/2026 04:03:44 UTC+07:00.

**H5. Dashboard bổ sung.** Có đủ bốn widget và legend; DLQ đã có giá trị 1 trong khoảng trước thử. Tại lúc chụp, biểu đồ Step Functions vẫn hiển thị ExecutionsFailed = 0; H6 ghi nhận lỗi ở ảnh khác. Không dùng H5 làm kết quả cuối của mọi execution.

![H5 - Dashboard bổ sung](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h5.png)

**H6. Step Functions.** Alarm order-monitoring-evidence-sfn-failed ở ALARM, biểu đồ ExecutionsFailed lên 1.

![H6 - Alarm Step Functions](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h6.png)

**H7. Lambda.** Alarm order-monitoring-evidence-lambda-errors ở ALARM, biểu đồ Errors lên 1. Riêng ảnh metric không chỉ ra nguyên nhân hoặc MessageId của invocation lỗi.

![H7 - Alarm Lambda Errors](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h7.png)

**H8. DLQ.** Đúng queue order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU chứa body not-json-mon56-20261114-040056-316515. Ảnh đồng thời có thông điệp cũ; Receive count không được dùng làm số lần Lambda xử lý chính xác vì thao tác nhận ở console cũng ảnh hưởng bộ đếm.

![H8 - Thông điệp thử trong DLQ](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h8.png)

**H9. Email ALARM Step Functions.** Email ghi datapoint 1 tại 13/11/2026 21:02 UTC và chuyển ALARM lúc 21:03:44 UTC. Đây là thông báo CloudWatch thực nhận. Chưa có mốc execution thất bại trong bộ ảnh này để tính độ trễ phát hiện.

![H9 - Email ALARM Step Functions](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h9.png)

**Email ALARM của Lambda.** Email từ CloudWatch ghi nhận datapoint Errors = 1 tại 13/11/2026 21:01:00 UTC, vượt ngưỡng 0. Alarm order-monitoring-evidence-lambda-errors chuyển từ OK sang ALARM lúc 21:02:21 UTC, tương ứng 14/11/2026 04:02:21 UTC+07:00. Email trong hộp thư xác nhận thông báo ALARM đã được nhận, phù hợp với trạng thái alarm trên CloudWatch. Mốc datapoint không phải thời điểm chính xác Lambda phát sinh lỗi, nên chưa dùng để tính độ trễ phát hiện lỗi hoặc độ trễ gửi email.

![Email ALARM Lambda](/images/5-Workshop/5.6-Monitoring/lambda-alarm-20261114-044826-a961f7/h10.png)

**H11. Email ALARM DLQ.** Email ghi datapoint 1 tại 13/11/2026 20:59 UTC, chuyển ALARM lúc 21:01:06 UTC. Datapoint này có trước lần gửi thông điệp mới trong H8; diễn giải là cảnh báo backlog đã tồn tại, không quy chuyển trạng thái này cho riêng thông điệp mới.

![H11 - Email cảnh báo DLQ tồn đọng](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h11.png)

## 5. Đánh giá và giới hạn

| Nội dung | Kết quả được minh chứng |
|---|---|
| Dashboard | Có bốn nhóm biểu đồ (H5) |
| Alarm Step Functions / Lambda | Cả hai được ghi nhận ALARM (H6, H7) |
| Thông điệp thử vào DLQ | Có body và queue trùng lần thử (H8) |
| Email ALARM Step Functions / DLQ | Đã nhận (H9, H11); DLQ gắn với backlog |
| Email ALARM Lambda | Đã nhận thông báo CloudWatch ALARM, khớp tên alarm và datapoint Errors = 1 |
| Phục hồi và độ trễ cảnh báo | Chưa đủ bằng chứng để kết luận hoặc tính thời gian |

Polling DLQ làm thông điệp tạm invisible, có thể giảm visible count dù chưa xử lý/xóa. OK với notBreaching không tự chứng minh ứng dụng khỏe. Giữ run.json, sent.json, alarms-before.json và dlq-before.json để đối chiếu; không suy ra tổng số đơn hay kết quả từng đơn chỉ từ metric.

## 6. Chi phí và dọn dẹp

Ba alarm và dashboard bổ sung có thể phát sinh phí. Sau khi lưu đủ bằng chứng, xóa riêng stack order-monitoring-evidence nếu không tiếp tục sử dụng. Stack ứng dụng và SNS dùng chung vẫn được giữ. Alarm order-orchestration-evidence-dlq-proof của mục 5.7 nằm ngoài stack này.

## Kết luận

Bộ minh chứng ghi nhận dashboard, alarm Step Functions/Lambda, thông điệp thử trong DLQ và email ALARM của cả ba alarm Step Functions, Lambda, DLQ. Cảnh báo DLQ phản ánh thông điệp tồn đọng trước lần thử. Bộ ảnh chưa đủ để kết luận phục hồi sau lỗi hoặc tính độ trễ cảnh báo từ thời điểm lỗi thực tế.