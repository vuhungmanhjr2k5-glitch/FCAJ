---
title: "Cấu hình metric và alarm"
weight: 2
chapter: false
pre: " <b> 5.6.2 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

| Alarm | Namespace / metric | Dimension | Statistic |
|---|---|---|---|
| order-monitoring-evidence-sfn-failed | AWS/States / ExecutionsFailed | StateMachineArn | Sum |
| order-monitoring-evidence-lambda-errors | AWS/Lambda / Errors | FunctionName | Sum |
| order-monitoring-evidence-dlq | AWS/SQS / ApproximateNumberOfMessagesVisible | QueueName của DLQ | Maximum |

Cả ba dùng ngưỡng **> 0**, Period **60 giây**, EvaluationPeriods **1**, DatapointsToAlarm **1**, TreatMissingData **notBreaching**. AlarmActions và OKActions trỏ tới SNS topic của ứng dụng.

Sum cộng số lỗi trong kỳ; Maximum phát hiện số thông điệp nhìn thấy trong DLQ. Metric SQS là xấp xỉ. Period 60 giây không bảo đảm email tới trong 60 giây. CloudWatch thường gửi thông báo khi chuyển trạng thái, không gửi lại cho mỗi lỗi trong lúc alarm vẫn ALARM.

## Tại sao mỗi alarm dùng một dimension khác

Tên metric chỉ xác định loại số liệu. `AWS/SQS` có thể chứa số liệu của nhiều hàng đợi, vì vậy alarm phải chọn đúng `QueueName` của DLQ. Nếu chọn tên hàng chính, mọi đơn chưa được xử lý đều có thể làm tăng bộ đếm. Với Lambda cần tên hàm vật lý. Với Step Functions cần toàn bộ ARN state machine. Thiếu một dimension chính xác có thể tạo một alarm hợp lệ về cấu hình nhưng không thu được datapoint phù hợp.

**Sum** trên `ExecutionsFailed` và `Errors` cộng các lần lỗi trong kỳ một phút. **Maximum** trên số tin visible lấy giá trị cao nhất được quan sát trong kỳ. Giả sử DLQ có một tin trong hai kỳ liên tiếp. Alarm có thể tiếp tục ở `ALARM` cả hai kỳ nhưng chỉ gửi thông báo chuyển trạng thái lần đầu. Hai datapoint cùng bằng 1 cũng có thể mô tả cùng một tin.

## Đọc template dùng cho tài khoản thực hành

Khối PowerShell ghi một template CloudFormation cục bộ. Người triển khai truyền định danh tài nguyên thật ở mục 5.6.3. Ba alarm cùng dùng topic nguồn cho AlarmActions và OKActions. Dashboard gắn Region vào từng widget để khi người dùng xem từ vùng khác, biểu đồ vẫn truy vấn đúng regional metric. Tên dashboard tạo từ tên stack giám sát để có thể tìm lại dễ dàng.

Trước khi chuyển bước, mở tệp vừa tạo và tìm bốn resource: ba alarm và một dashboard. Kiểm tra từng cặp namespace và dimension theo bảng đầu trang. Nếu chỉnh period hoặc cách xử lý missing data, ghi thay đổi ngay tại đây. Những thông số này quyết định cách đọc email khi chạy thực nghiệm.

### Đọc một kỳ đánh giá giả định

Giả sử số tin visible cao nhất của DLQ là 0 ở kỳ thứ nhất, 1 ở kỳ thứ hai và 1 ở kỳ thứ ba. Với `GreaterThanThreshold` và ngưỡng 0, kỳ thứ hai đủ điều kiện báo lỗi. Nếu alarm đã ALARM từ kỳ thứ hai, kỳ thứ ba không tạo thêm một lần chuyển vào ALARM. Một tin lỗi khác đến lúc này vẫn cần xử lý dù người trực có thể chỉ thấy một email. Trạng thái trước thử vì vậy là một trường quan trọng trong sổ quan sát.

Với Lambda và Step Functions, hãy đặt đúng metric cạnh đúng loại sự cố. Tin không phải JSON có thể làm một message gây nhiều invocation lỗi vì SQS giao lại. Ngược lại, ba lần SES task thất bại trong một execution có thể chỉ tạo một lần `ExecutionsFailed` khi workflow kết thúc. Giá trị 1 từ hai namespace không đếm cùng đơn vị nghiệp vụ.

### Ghi template giám sát dùng cho phần 5.6.3

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

Bốn widget dưới đây dùng chung Region. Mỗi dòng trong mảng metrics chỉ rõ namespace, metric và dimension. Chạy hai khối tiếp theo trong cùng cửa sổ PowerShell.

```powershell
$dashboardJson = @'
{
  "widgets": [
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Maximum",
        "metrics": [
              ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "${SourceQueueName}", {"label": "Source queue"}],
              ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "${DlqName}", {"label": "DLQ"}]
        ],
        "period": 60,
        "title": "SQS: visible messages",
        "view": "timeSeries"
      },
      "y": 0,
      "type": "metric",
      "x": 0,
      "height": 6
    },
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Maximum",
        "metrics": [
              ["AWS/SQS", "ApproximateAgeOfOldestMessage", "QueueName", "${SourceQueueName}"]
        ],
        "period": 60,
        "title": "SQS: oldest source message (seconds)",
        "view": "timeSeries"
      },
      "y": 0,
      "type": "metric",
      "x": 12,
      "height": 6
    },
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Sum",
        "metrics": [
              ["AWS/States", "ExecutionsStarted", "StateMachineArn", "${MachineArn}"],
              ["AWS/States", "ExecutionsSucceeded", "StateMachineArn", "${MachineArn}"],
              ["AWS/States", "ExecutionsFailed", "StateMachineArn", "${MachineArn}"]
        ],
        "period": 60,
        "title": "Step Functions: executions",
        "view": "timeSeries"
      },
      "y": 6,
      "type": "metric",
      "x": 0,
      "height": 6
    },
    {
      "width": 12,
      "properties": {
        "legend": {
          "position": "bottom"
        },
        "region": "${AWS::Region}",
        "stat": "Sum",
        "metrics": [
              ["AWS/Lambda", "Invocations", "FunctionName", "${WorkerName}"],
              ["AWS/Lambda", "Errors", "FunctionName", "${WorkerName}"]
        ],
        "period": 60,
        "title": "Lambda: invocations and errors",
        "view": "timeSeries"
      },
      "y": 6,
      "type": "metric",
      "x": 12,
      "height": 6
    }
  ]
}
'@
$null = $dashboardJson | ConvertFrom-Json -ErrorAction Stop
```

Khối thứ hai đặt ba alarm, ghép JSON của dashboard vào Fn::Sub và ghi tệp. Chuỗi chứa tham chiếu tài nguyên được giữ nguyên để CloudFormation thay thế khi deploy.

```powershell
if ([string]::IsNullOrWhiteSpace($dashboardJson)) { throw 'Run the dashboard block above first.' }
$definitionJson = @'
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Order workshop monitoring for a selected application stack",
  "Parameters": {
    "DlqName": {
      "Type": "String"
    },
    "MachineArn": {
      "Type": "String"
    },
    "SourceQueueName": {
      "Type": "String"
    },
    "SourceStack": {
      "Type": "String"
    },
    "TopicArn": {
      "Type": "String"
    },
    "WorkerName": {
      "Type": "String"
    }
  },
  "Resources": {
    "OrderDashboard": {
      "Properties": {
        "DashboardName": {
          "Fn::Sub": "${AWS::StackName}-dashboard"
        },
        "DashboardBody": {
          "Fn::Sub": ""
        }
      },
      "Type": "AWS::CloudWatch::Dashboard"
    },
    "SfnFailedAlarm": {
      "Properties": {
        "AlarmName": {
          "Fn::Sub": "${AWS::StackName}-sfn-failed"
        },
        "ComparisonOperator": "GreaterThanThreshold",
        "OKActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "AlarmActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "Dimensions": [
          {
            "Value": {
              "Ref": "MachineArn"
            },
            "Name": "StateMachineArn"
          }
        ],
        "EvaluationPeriods": 1,
        "Namespace": "AWS/States",
        "AlarmDescription": {
          "Fn::Sub": "Monitoring ${SourceStack}"
        },
        "TreatMissingData": "notBreaching",
        "MetricName": "ExecutionsFailed",
        "Statistic": "Sum",
        "Threshold": 0,
        "DatapointsToAlarm": 1,
        "Period": 60
      },
      "Type": "AWS::CloudWatch::Alarm"
    },
    "DlqAlarm": {
      "Properties": {
        "AlarmName": {
          "Fn::Sub": "${AWS::StackName}-dlq"
        },
        "ComparisonOperator": "GreaterThanThreshold",
        "OKActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "AlarmActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "Dimensions": [
          {
            "Value": {
              "Ref": "DlqName"
            },
            "Name": "QueueName"
          }
        ],
        "EvaluationPeriods": 1,
        "Namespace": "AWS/SQS",
        "AlarmDescription": {
          "Fn::Sub": "Monitoring ${SourceStack}"
        },
        "TreatMissingData": "notBreaching",
        "MetricName": "ApproximateNumberOfMessagesVisible",
        "Statistic": "Maximum",
        "Threshold": 0,
        "DatapointsToAlarm": 1,
        "Period": 60
      },
      "Type": "AWS::CloudWatch::Alarm"
    },
    "LambdaErrorAlarm": {
      "Properties": {
        "AlarmName": {
          "Fn::Sub": "${AWS::StackName}-lambda-errors"
        },
        "ComparisonOperator": "GreaterThanThreshold",
        "OKActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "AlarmActions": [
          {
            "Ref": "TopicArn"
          }
        ],
        "Dimensions": [
          {
            "Value": {
              "Ref": "WorkerName"
            },
            "Name": "FunctionName"
          }
        ],
        "EvaluationPeriods": 1,
        "Namespace": "AWS/Lambda",
        "AlarmDescription": {
          "Fn::Sub": "Monitoring ${SourceStack}"
        },
        "TreatMissingData": "notBreaching",
        "MetricName": "Errors",
        "Statistic": "Sum",
        "Threshold": 0,
        "DatapointsToAlarm": 1,
        "Period": 60
      },
      "Type": "AWS::CloudWatch::Alarm"
    }
  },
  "Outputs": {
    "DashboardName": {
      "Value": {
        "Ref": "OrderDashboard"
      }
    },
    "SourceStack": {
      "Value": {
        "Ref": "SourceStack"
      }
    }
  }
}
'@
$definition = $definitionJson | ConvertFrom-Json -ErrorAction Stop
$definition.Resources.OrderDashboard.Properties.DashboardBody.'Fn::Sub' = $dashboardJson
$content = $definition | ConvertTo-Json -Depth 30
$path = Join-Path $PWD.Path 'monitoring.template.json'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

Template tĩnh của lần thực nghiệm gốc được lưu cùng bộ tài liệu.

[monitoring.template.json](/downloads/5.6-Monitoring/evidence-20261114-043035-985e31/monitoring.template.json)
