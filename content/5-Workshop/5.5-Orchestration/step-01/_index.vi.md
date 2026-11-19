---
title: "Thay toàn bộ template.yaml"
weight: 1
chapter: false
pre: " <b> 5.5.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Đọc định nghĩa như theo dõi một đơn

Bắt đầu tại `StartAt` rồi đi theo `Next` ở từng trạng thái. Dữ liệu đầu vào chứa ít nhất `orderId` và `customerEmail`. Tại `SendCustomerEmail`, hàm nội tại `States.Array` tạo danh sách người nhận từ một địa chỉ. `ResultPath: null` giữ đầu vào ban đầu sau khi SES trả lời, nhờ đó `MarkDone` vẫn có mã đơn để cập nhật đúng item trong DynamoDB. Nếu thay toàn bộ input bằng kết quả của SES, bước kế tiếp có thể không còn tìm thấy khóa.

Ở đường lỗi, Catch đặt chi tiết lỗi vào `$.error` và vẫn giữ `orderId`. `MarkFailed` ghi trạng thái, `NotifyAdminFailure` gửi nội dung có mã đơn, sau đó `OrderFailed` kết thúc execution. Khi đọc lịch sử, hãy xem trạng thái đầu tiên thất bại và trạng thái cuối cùng đã đi qua. Nếu bản thân bước ghi `FAILED` lỗi quyền, chuỗi xử lý có thể dừng trước SNS.

## Tính khoảng chờ trước khi kết luận

Với `IntervalSeconds` bằng 2, `BackoffRate` bằng 2 và `MaxAttempts` bằng 3, một lỗi kéo dài có thể dẫn đến lần gọi đầu cùng ba lần thử lại. Ba khoảng chờ là 2, 4 và 8 giây, tổng 14 giây, chưa kể thời gian gọi dịch vụ và chuyển state. Nếu lần gọi thứ hai thành công, các khoảng chờ tiếp theo sẽ không diễn ra. Khi ghi báo cáo, phân biệt “ba lần thử lại được cấu hình” và số lần thực sự quan sát trong history.

Template thêm hai tham số khác vai trò. `SenderEmail` dùng cho thư xác nhận đơn, còn `AdminEmail` đăng ký nhận thông báo từ topic. Kiểm tra lại hai giá trị trước deploy để tránh trường hợp workflow hoạt động nhưng cảnh báo đang đến hộp thư khác.

## Ghi toàn bộ `template.yaml`

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

```powershell
$content = @'
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Order workshop - xu ly don hang event-driven

Parameters:
  SenderEmail:
    Type: String
    Description: Email da xac minh trong SES, dung lam nguoi gui
  AdminEmail:
    Type: String
    Description: Email nhan canh bao khi xu ly don that bai

Resources:
  OrdersDLQ:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600

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

  OrderTopic:
    Type: AWS::SNS::Topic
    Properties:
      Subscription:
        - Protocol: email
          Endpoint: !Ref AdminEmail

  OrderStateMachine:
    Type: AWS::Serverless::StateMachine
    Properties:
      DefinitionSubstitutions:
        TableName: !Ref OrdersTable
        TopicArn: !Ref OrderTopic
        SenderEmail: !Ref SenderEmail
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - SNSPublishMessagePolicy:
            TopicName: !GetAtt OrderTopic.TopicName
        - Statement:
            - Effect: Allow
              Action: ses:SendEmail
              Resource: "*"
      Definition:
        StartAt: SendCustomerEmail
        States:
          SendCustomerEmail:
            Type: Task
            Resource: arn:aws:states:::aws-sdk:sesv2:sendEmail
            Parameters:
              FromEmailAddress: ${SenderEmail}
              Destination:
                ToAddresses.$: States.Array($.customerEmail)
              Content:
                Simple:
                  Subject:
                    Data.$: "States.Format('Xac nhan don hang {}', $.orderId)"
                  Body:
                    Text:
                      Data.$: "States.Format('Cam on ban. Don hang {} da duoc ghi nhan.', $.orderId)"
            ResultPath: null
            Retry:
              - ErrorEquals: [States.ALL]
                IntervalSeconds: 2
                MaxAttempts: 3
                BackoffRate: 2
            Catch:
              - ErrorEquals: [States.ALL]
                ResultPath: $.error
                Next: MarkFailed
            Next: MarkDone

          MarkDone:
            Type: Task
            Resource: arn:aws:states:::dynamodb:updateItem
            Parameters:
              TableName: ${TableName}
              Key:
                orderId:
                  S.$: $.orderId
              UpdateExpression: "SET #s = :v"
              ExpressionAttributeNames:
                "#s": status
              ExpressionAttributeValues:
                ":v":
                  S: DONE
            End: true

          MarkFailed:
            Type: Task
            Resource: arn:aws:states:::dynamodb:updateItem
            Parameters:
              TableName: ${TableName}
              Key:
                orderId:
                  S.$: $.orderId
              UpdateExpression: "SET #s = :v"
              ExpressionAttributeNames:
                "#s": status
              ExpressionAttributeValues:
                ":v":
                  S: FAILED
            ResultPath: null
            Next: NotifyAdminFailure

          NotifyAdminFailure:
            Type: Task
            Resource: arn:aws:states:::sns:publish
            Parameters:
              TopicArn: ${TopicArn}
              Subject: Canh bao - xu ly don hang that bai
              Message.$: "States.Format('Don hang {} xu ly that bai. Kiem tra execution trong Step Functions.', $.orderId)"
            ResultPath: null
            Next: OrderFailed

          OrderFailed:
            Type: Fail
            Error: OrderProcessingFailed
            Cause: Khong gui duoc email xac nhan

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
          STATE_MACHINE_ARN: !Ref OrderStateMachine
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - StepFunctionsExecutionPolicy:
            StateMachineName: !GetAtt OrderStateMachine.Name
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
  StateMachineArn:
    Value: !Ref OrderStateMachine
'@
$path = Join-Path $PWD.Path 'template.yaml'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

> Quyền `ses:SendEmail` để `Resource: "*"` cho đơn giản trong môi trường thử
> nghiệm. Với môi trường thật, hãy giới hạn theo identity của người gửi.
