---
title: "Cập nhật template.yaml"
weight: 1
chapter: false
pre: " <b> 5.4.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Ba mối liên kết cần đọc trong template

`OrdersTable` dùng `orderId` kiểu chuỗi làm partition key. Hai lần ghi cùng mã sẽ tác động đến cùng một item. Điều kiện chống trùng được thêm trong mã Lambda ở bước sau. Riêng thiết kế khóa chỉ xác định item nào là đích của thao tác. Chế độ `PAY_PER_REQUEST` giúp bài thử với số đơn ít có thể dùng bảng mà không phải chọn công suất đọc/ghi cố định trước.

Trong `ProcessOrderFn`, `TABLE_NAME` lấy tên bảng vật lý từ CloudFormation. Policy của hàm cũng tham chiếu chính bảng đó. Nếu sửa một nơi mà quên nơi còn lại, Lambda có thể gọi đúng tên nhưng bị từ chối quyền, hoặc có quyền nhưng ứng dụng lại dùng sai bảng. `Handler: index.handler` tương ứng với hàm export tên `handler` trong `src/index.mjs`.

## Hình dung một batch có một tin

SQS event source mapping gom thông điệp thành batch trước khi gọi Lambda. Ở đây `BatchSize` bằng 1 nên một lỗi parse chỉ làm lần xử lý của một thông điệp thất bại. Timeout Lambda là 10 giây và thời gian ẩn của queue là 60 giây, phù hợp mức sáu lần timeout khi batching window bằng 0. Khi thay đổi timeout cho bài thực hành khác, hãy tính lại cặp giá trị này.

Sau khi ghi tệp, đọc mục `Outputs` ở cuối. Ngoài hai URL cũ, phải có `TableName` để các lệnh kiểm thử tra đúng bảng. Hãy tạo mã Lambda ở mục 5.4.2 trước khi build template mới.

## Ghi toàn bộ `template.yaml`

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

```powershell
$content = @'
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Order workshop - SQS va DLQ

Resources:
  OrdersDLQ:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600   # giu tin nhan loi 14 ngay

  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 60
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrdersDLQ.Arn
        maxReceiveCount: 3

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: orderId
          AttributeType: S
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH

  ProcessOrderFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs22.x
      CodeUri: src/
      Timeout: 10
      Environment:
        Variables:
          TABLE_NAME: !Ref OrdersTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
      Events:
        FromSqs:
          Type: SQS
          Properties:
            Queue: !GetAtt OrdersQueue.Arn
            BatchSize: 1

Outputs:
  QueueUrl:
    Value: !Ref OrdersQueue
  DlqUrl:
    Value: !Ref OrdersDLQ
  TableName:
    Value: !Ref OrdersTable
'@
$path = Join-Path $PWD.Path 'template.yaml'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

> `VisibilityTimeout` của hàng đợi (60 giây) lớn hơn nhiều lần `Timeout` của
> Lambda (10 giây). Đây là cấu hình khuyến nghị khi Lambda đọc từ SQS.
