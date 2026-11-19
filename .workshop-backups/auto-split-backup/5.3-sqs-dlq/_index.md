---
title: "Queues and DLQ"
weight: 3
chapter: false
pre: " <b> 5.3 </b> "
---
## Objective

Create a main queue that receives orders and a dead-letter queue (**DLQ**)
to keep messages that fail processing, instead of losing them.

Three concepts to remember:

- **Visibility timeout**: the time a message stays hidden after a consumer receives it. If the consumer does not delete the message within this time, the message becomes visible again so it can be retried.
- **maxReceiveCount**: the number of times a message can be received without being deleted before it is moved to the DLQ.
- **DLQ**: a queue that holds failed messages so they can be inspected and reprocessed.

#### 1. Create the template.yaml file

In the `order-system` folder, create a file named `template.yaml`:

```yaml
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
```

#### 2. Deploy

```powershell
sam build
sam deploy --guided
```

Answer the prompts as follows:

| Prompt | Answer |
|---|---|
| Stack Name | `order-system` |
| AWS Region | `ap-southeast-1` |
| Confirm changes before deploy | `N` |
| Allow SAM CLI IAM role creation | `Y` |
| Disable rollback | `N` |
| Save arguments to samconfig.toml | `Y` |

When it finishes, SAM prints `QueueUrl` and `DlqUrl`.

![Deploy result](/images/5-Workshop/5.3-SQS-DLQ/deploy-result.png)

#### 3. Verify in the Console

1. Go to **Amazon SQS** → **Queues**.
2. You will see two queues whose names start with `order-system-`.
3. Open the main queue → **Dead-letter queue** tab: the dead-letter queue is attached, with **Maximum receives = 3**.

![SQS queues](/images/5-Workshop/5.3-SQS-DLQ/queues.png)

#### 4. Test a message falling into the DLQ

Store the two queue URLs in variables:

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue" --output text
$dlq = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='DlqUrl'].OutputValue" --output text
```

Send a test message:

```powershell
aws sqs send-message --queue-url $q --message-body "test-dlq"
```

Receive the message several times **without deleting it** (the `--visibility-timeout 0` parameter makes the message visible again immediately):

```powershell
1..4 | ForEach-Object {
  aws sqs receive-message --queue-url $q --visibility-timeout 0 --wait-time-seconds 2
  Start-Sleep -Seconds 1
}
```

The first three attempts return the message, and the fourth returns nothing. Wait about a minute, then check the DLQ:

```powershell
aws sqs get-queue-attributes --queue-url $dlq --attribute-names ApproximateNumberOfMessages
```

Or go to the Console → the DLQ queue → **Send and receive messages** → **Poll for messages**.

![Message in the DLQ](/images/5-Workshop/5.3-SQS-DLQ/dlq-message.png)

> SQS metrics are approximate and may be delayed by several tens of seconds.

## Verify the result

- There are two queues, and the main queue points to the DLQ with `maxReceiveCount = 3`.
- A message received more than 3 times without being deleted ends up in the DLQ.