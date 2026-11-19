---
title: "Cập nhật template.yaml"
weight: 1
chapter: false
pre: " <b> 5.7.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Giữ cùng hạ tầng bên dưới hai đường API

Cả hai endpoint nhận cùng dạng body. `/sync` giao cho một Lambda thực hiện ghi bảng, gửi SES và cập nhật kết quả trong cùng request. `/async` giao cho Lambda tiếp nhận đưa body vào queue rồi trả lời. Bảng, địa chỉ người gửi và quy trình phía sau dùng lại tài nguyên của các chương trước. Khi đo trên cùng tài khoản, hãy giữ cùng Region và mỗi nhóm đơn dùng một tiền tố riêng để kết quả hai nhánh không lẫn vào nhau.

Template dưới đây là phiên bản hoàn chỉnh sau khi ghép các thay đổi vào bản mẫu 5.5.1. Nó giữ logical ID của queue, bảng, worker và state machine. Nếu bạn tự thêm tài nguyên vào template riêng, đối chiếu trước khi thay tệp. Các alarm ở 5.6 được quản lý ở stack giám sát riêng. Lần cập nhật này thêm hai hàm HTTP và đầu ra `ApiUrl` vào stack ứng dụng.

## Hiểu các số đo mới trên item

`createdAt` là thời điểm ứng dụng ghi nhận đầu vào. `doneAt` ở nhánh bất đồng bộ lấy thời điểm bước `MarkDone` bắt đầu. `emailCount` được tăng cùng thao tác cập nhật khi đi qua bước thành công. Bộ đếm này đo hiệu ứng ghi nhận của ứng dụng. Nó không thay thế nhật ký chuyển phát của SES hay quan sát hộp thư thật.

Các lần retry dùng backoff và full jitter để nhiều execution bị giới hạn gửi thư có thể thử lại ở các thời điểm khác nhau. Với sáu lần thử lại, các mức trần khoảng chờ là 2, 4, 8, 16, 32 và 64 giây. Tổng trần thời gian **chờ theo lịch** là 126 giây, chưa tính thời gian gọi dịch vụ. Dùng history thực tế khi tính tổng thời gian xử lý một đơn lỗi.

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

  ApiKey:
    Type: String
    NoEcho: true
    MinLength: 16
    Description: Khoa bi mat cho header x-api-key

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
                MaxAttempts: 6
                BackoffRate: 2
                JitterStrategy: FULL
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
              UpdateExpression: "SET #s = :v, doneAt = :t ADD emailCount :one"
              ExpressionAttributeNames:
                "#s": status
              ExpressionAttributeValues:
                ":v":
                  S: DONE
                ":t":
                  S.$: $$.State.EnteredTime
                ":one":
                  N: "1"
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

  SyncOrderFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: sync-order.handler
      Runtime: nodejs22.x
      CodeUri: src/
      MemorySize: 256
      Timeout: 15
      Environment:
        Variables:
          TABLE_NAME: !Ref OrdersTable
          SENDER_EMAIL: !Ref SenderEmail
          API_KEY: !Ref ApiKey
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - Statement:
            - Effect: Allow
              Action: ses:SendEmail
              Resource: "*"
      Events:
        Post:
          Type: HttpApi
          Properties:
            Path: /sync
            Method: POST

  AsyncOrderFn:
    Type: AWS::Serverless::Function
    Properties:
      Handler: async-order.handler
      Runtime: nodejs22.x
      CodeUri: src/
      MemorySize: 256
      Timeout: 15
      Environment:
        Variables:
          QUEUE_URL: !Ref OrdersQueue
          API_KEY: !Ref ApiKey
      Policies:
        - SQSSendMessagePolicy:
            QueueName: !GetAtt OrdersQueue.QueueName
      Events:
        Post:
          Type: HttpApi
          Properties:
            Path: /async
            Method: POST

Outputs:
  QueueUrl:
    Value: !Ref OrdersQueue
  DlqUrl:
    Value: !Ref OrdersDLQ
  TableName:
    Value: !Ref OrdersTable
  StateMachineArn:
    Value: !Ref OrderStateMachine
  ApiUrl:
    Value: !Sub "https://${ServerlessHttpApi}.execute-api.${AWS::Region}.amazonaws.com"
'@
$path = Join-Path $PWD.Path 'template.yaml'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```
