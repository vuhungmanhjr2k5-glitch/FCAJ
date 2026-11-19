---
title: "Update template.yaml"
weight: 1
chapter: false
pre: " <b> 5.7.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Keep both API paths on the same underlying resources

Both endpoints accept the same payload shape. `/sync` asks a Lambda function to update the table, submit mail and record completion inside the request. `/async` asks an intake function to enqueue the body and respond. The table, sender and downstream workflow reuse earlier resources. Keep the same account and Region, and assign distinct ID prefixes to each group so their results stay separable.

The following template is the complete 5.5.1 example with the API changes applied. It preserves the queue, table, worker and state machine logical IDs. Compare local customizations before replacing an edited template. The alarms from 5.6 live in a separate monitoring stack. This application update adds the HTTP functions and `ApiUrl` output.

## Interpret the new item fields

`createdAt` marks the application's intake time. On the async path `doneAt` comes from entry into `MarkDone`. The application increments `emailCount` with its successful completion update. This is an application-side count. A delivery event or a real mailbox observation answers the separate question of mail arrival.

Exponential backoff with full jitter gives throttled executions different opportunities to retry. With six retries the delay ceilings are 2, 4, 8, 16, 32 and 64 seconds. Their total scheduled-wait ceiling is 126 seconds before service call duration. Use the actual execution history for an individual failure.

## Write the complete `template.yaml`

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

```powershell
$content = @'
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Order workshop - event-driven order processing

Parameters:
  SenderEmail:
    Type: String
    Description: Email verified in SES, used as the sender
  AdminEmail:
    Type: String
    Description: Email that receives alerts when order processing fails

  ApiKey:
    Type: String
    NoEcho: true
    MinLength: 16
    Description: Secret key for the x-api-key header

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
                    Data.$: "States.Format('Order confirmation {}', $.orderId)"
                  Body:
                    Text:
                      Data.$: "States.Format('Thank you. Your order {} has been received.', $.orderId)"
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
              Subject: Alert - order processing failed
              Message.$: "States.Format('Order {} failed to process. Check the execution in Step Functions.', $.orderId)"
            ResultPath: null
            Next: OrderFailed

          OrderFailed:
            Type: Fail
            Error: OrderProcessingFailed
            Cause: Could not send the confirmation email

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
