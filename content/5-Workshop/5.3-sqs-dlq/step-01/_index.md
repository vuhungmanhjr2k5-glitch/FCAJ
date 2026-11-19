---
title: "Create the template.yaml file"
weight: 1
chapter: false
pre: " <b> 5.3.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

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

Outputs:
  QueueUrl:
    Value: !Ref OrdersQueue
  DlqUrl:
    Value: !Ref OrdersDLQ
'@
$path = Join-Path $PWD.Path 'template.yaml'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Read the dead-letter queue before the source queue

`OrdersDLQ` sets retention to 1,209,600 seconds, or 14 days. For a Standard queue, retention still depends on the original enqueue time after transfer to the DLQ. A longer retention gives an operator time to investigate. The transfer itself does not reset the full retention interval.

In `OrdersQueue`, `deadLetterTargetArn` refers to the ARN of the `OrdersDLQ` resource in this template. The physical queue names can differ between accounts. Linking by logical ID preserves the relationship on each deployment. The outputs expose URLs because SQS uses a queue URL for send, receive and attribute operations.

## Trace a delivery attempt

Suppose a consumer receives a message at 10:00:00. With a 60-second visibility timeout, other consumers ordinarily cannot see that message during the interval. Deleting it after processing completes the delivery. Leaving it undeleted makes redelivery possible when the interval expires. An actual receive increases the receive count. Waiting beside an idle queue does not make a message move toward the DLQ.

## Read the saved file before building

```powershell
Get-Item -LiteralPath .\template.yaml -ErrorAction Stop | Select-Object FullName, Length
Select-String -LiteralPath .\template.yaml -Pattern '^  (OrdersQueue|OrdersDLQ|QueueUrl|DlqUrl):'
```

Locate the two resource IDs and two output names at their intended indentation levels. If the file is missing, return to the application directory from 5.2.6. This part of the exercise is local and can be checked before provisioning SQS.
