---
title: "Replace the entire template.yaml"
weight: 1
chapter: false
pre: " <b> 5.5.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Trace one order through the state definition

Begin at `StartAt` and follow each `Next`. Input includes at least `orderId` and `customerEmail`. In `SendCustomerEmail`, `States.Array` creates the recipient list from one address. `ResultPath: null` retains the input after the SES response, leaving the order ID available to `MarkDone`. Replacing the input with the SES response could remove the key the next state needs.

On the error path, Catch places the error under `$.error` while retaining the order ID. `MarkFailed` updates the item, `NotifyAdminFailure` publishes the ID and `OrderFailed` terminates the execution. Inspect both the first failing state and the last state reached. A denial in the failed-status update itself can end the flow before the SNS publish.

## Work out the wait before interpreting the result

An interval of 2 seconds, backoff of 2 and three retry attempts allow the original call plus three further calls. The waits are 2, 4 and 8 seconds, a total of 14 seconds before service and transition time. If the second call succeeds, the later waits never occur. Record configured retry capacity separately from the attempts observed in history.

The template also introduces two email parameters. `SenderEmail` sends order confirmations. `AdminEmail` subscribes to the administrative topic. Checking both values before deployment avoids searching the wrong inbox later.

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

> The `ses:SendEmail` permission uses `Resource: "*"` for simplicity in a test
> environment. In a real environment, restrict it to the sender's identity.
