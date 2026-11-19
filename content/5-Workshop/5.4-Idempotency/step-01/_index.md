---
title: "Update template.yaml"
weight: 1
chapter: false
pre: " <b> 5.4.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Read the three bindings in the template

`OrdersTable` uses a string `orderId` as its partition key. Two writes for the same purchase target the same item. The worker will supply the conditional write in the next step. The key alone identifies the destination of an operation. With `PAY_PER_REQUEST` the small lab can use the table without sizing read and write capacity in advance.

In `ProcessOrderFn`, `TABLE_NAME` receives the physical table name from CloudFormation. The function policy references the same table. An inconsistent edit could leave the worker authorized for one table while its code addresses another. `Handler: index.handler` resolves to the exported `handler` in `src/index.mjs`.

## Picture a one-message batch

The SQS event source mapping calls Lambda with a batch. Here `BatchSize` is one, so a parse error fails processing for one message. The 10-second function timeout and 60-second queue visibility timeout meet the six-times guideline with a zero batching window. Revisit both values if adapting the worker timeout for another exercise.

Read the final `Outputs` section after writing the file. Alongside both existing URLs, `TableName` lets the tests locate the deployed table. Create the handler in 5.4.2 before building this new version.

## Write the complete `template.yaml`

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

```powershell
$content = @'
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Order workshop - SQS and DLQ

Resources:
  OrdersDLQ:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600   # keep failed messages for 14 days

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

> The queue's `VisibilityTimeout` (60 seconds) is several times larger than the
> Lambda `Timeout` (10 seconds). This is the recommended configuration when
> Lambda reads from SQS.
