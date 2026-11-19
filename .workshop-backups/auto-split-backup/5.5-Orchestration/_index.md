---
title: "Orchestration and Notifications"
weight: 5
chapter: false
pre: " <b> 5.5 </b> "
---
## Objective

Instead of letting a single Lambda function do everything, we use **AWS Step Functions** to
orchestrate the order-processing workflow. Each step has clear **Retry** and
**Catch** behavior, the state of every order is recorded, and administrators
are notified immediately through **Amazon SNS** when an order fails.

Workflow:

1. Lambda receives a message from SQS, records it in DynamoDB (`RECEIVED`), and starts the state machine.
2. The state machine sends a confirmation email to the customer using **Amazon SES** (up to 3 retries).
3. On success: update the status to `DONE`.
4. On failure: update the status to `FAILED`, send an alert through SNS, and end in a failed state.

![Workflow diagram](/images/5-Workshop/5.5-Orchestration/workflow.png)

#### 1. Replace the entire template.yaml

```yaml
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
```

> The `ses:SendEmail` permission uses `Resource: "*"` for simplicity in a test
> environment. In a real environment, restrict it to the sender's identity.

#### 2. Replace the entire src/index.mjs

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  PutCommand,
  GetCommand,
} from "@aws-sdk/lib-dynamodb";
import { SFNClient, StartExecutionCommand } from "@aws-sdk/client-sfn";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const sfn = new SFNClient({});

export const handler = async (event) => {
  for (const record of event.Records) {
    const order = JSON.parse(record.body); // invalid JSON -> error -> retry -> DLQ

    // Step 1: write to the ledger; each orderId may only be written once
    try {
      await db.send(new PutCommand({
        TableName: process.env.TABLE_NAME,
        Item: { orderId: order.orderId, status: "RECEIVED", createdAt: Date.now() },
        ConditionExpression: "attribute_not_exists(orderId)",
      }));
    } catch (e) {
      if (e.name !== "ConditionalCheckFailedException") throw e;

      const { Item } = await db.send(new GetCommand({
        TableName: process.env.TABLE_NAME,
        Key: { orderId: order.orderId },
      }));
      if (Item?.status !== "RECEIVED") {
        console.log("Duplicate order, skipping:", order.orderId);
        continue;
      }
      // Still in RECEIVED state: the previous attempt failed to start the workflow -> retry
    }

    // Step 2: start the workflow. The execution name = orderId, so it cannot run twice.
    try {
      await sfn.send(new StartExecutionCommand({
        stateMachineArn: process.env.STATE_MACHINE_ARN,
        name: order.orderId,
        input: JSON.stringify(order),
      }));
      console.log("Workflow started:", order.orderId);
    } catch (e) {
      if (e.name === "ExecutionAlreadyExists") {
        console.log("Workflow already exists, skipping:", order.orderId);
        continue;
      }
      throw e;
    }
  }
};
```

> Because `orderId` is used as the execution name, it should only contain letters,
> digits, `-`, and `_` (up to 80 characters).

#### 3. Deploy

```powershell
sam build
sam deploy --guided
```

Keep the same answers as last time. When asked for the two parameters, enter:

| Parameter | Value |
|---|---|
| SenderEmail | The email verified in section 5.2 |
| AdminEmail | The email that receives alerts (can be the same email as above) |

![Deploy result](/images/5-Workshop/5.5-Orchestration/deploy-result.png)

**Confirm the SNS subscription:** AWS sends an email titled *AWS Notification -
Subscription Confirmation* to `AdminEmail`. Open the email and click **Confirm
subscription**, otherwise you will not receive alerts.

![Confirm SNS](/images/5-Workshop/5.5-Orchestration/sns-confirm.png)

#### 4. Testing

Get the variables you need:

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue" --output text
$table = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue" --output text
$email = "your-verified-email@example.com"
```

**Case 1: successful order**

```powershell
@{ orderId = "B001"; customerEmail = $email } | ConvertTo-Json -Compress | Out-File -Encoding ascii msg-b001.json
aws sqs send-message --queue-url $q --message-body file://msg-b001.json
```

Verify:

1. Your inbox receives the email *Order confirmation B001*.
2. Go to **Step Functions** → **State machines** → select the state machine → the execution named `B001` → status **Succeeded**. Open **Graph view** to see the steps in green.
3. `aws dynamodb scan --table-name $table` shows `B001` with `status = DONE`.

**Order B001 execution completed successfully**

![Execution B001 has status Succeeded](/images/5-Workshop/5.5-Orchestration/b001-execution-succeeded.png)

**Workflow execution steps**

![Graph view showing successful SendCustomerEmail and MarkDone steps](/images/5-Workshop/5.5-Orchestration/b001-workflow-graph.png)

Both `SendCustomerEmail` and `MarkDone` completed successfully.
The failure-handling branch was not executed in this run.

**Order state in DynamoDB**

![DynamoDB records order B001 with status DONE](/images/5-Workshop/5.5-Orchestration/b001-dynamodb-done.png)

The `B001` record contains `status = DONE`, a `doneAt` field,
and `emailCount = 1`.

**Case 2: duplicate order**

```powershell
aws sqs send-message --queue-url $q --message-body file://msg-b001.json
sam logs --stack-name order-system --name ProcessOrderFn --tail
```

The log shows `Duplicate order, skipping: B001`, and no second email is sent.
Press `Ctrl+C` to exit.

![Successful execution](/images/5-Workshop/5.5-Orchestration/execution-success.png)

**Case 3: email sending failure**

Use an address that is **not verified** in SES. Because the account is in
sandbox mode, SES will reject it:

```powershell
@{ orderId = "B002"; customerEmail = "unverified@example.com" } | ConvertTo-Json -Compress | Out-File -Encoding ascii msg-b002.json
aws sqs send-message --queue-url $q --message-body file://msg-b002.json
```

Wait about 20 to 30 seconds (3 retries spaced 2, 4, and 8 seconds apart), then check:

1. The `B002` execution is in **Failed** status. In Graph view, the `SendCustomerEmail` step shows an error, and the flow goes through `MarkFailed` and `NotifyAdminFailure`.
2. The alert email *Alert - order processing failed* arrives at `AdminEmail`.
3. DynamoDB: `B002` has `status = FAILED`.

![Failed execution](/images/5-Workshop/5.5-Orchestration/execution-failed.png)
![Alert email](/images/5-Workshop/5.5-Orchestration/alert-email.png)

## Verify the results

| Case | Expected result |
|---|---|
| Valid order | Email sent to the customer, execution Succeeded, status `DONE` |
| Duplicate submission | No second email, log reports a duplicate |
| Email cannot be sent | 3 retries, execution Failed, status `FAILED`, admin receives an alert |

## Troubleshooting

- **Not receiving the order confirmation email**: check that both `SenderEmail` and the customer email are *Verified* in SES, and check your spam folder.
- **Not receiving the alert**: check whether you clicked *Confirm subscription* in SNS (Console → SNS → Subscriptions, the status must be *Confirmed*).
- **No execution at all**: check the `ProcessOrderFn` logs; if there is a permissions error, review the `Policies` section in the template.
- **Execution reports an `AccessDenied` error at SES**: check that the state machine's role has `ses:SendEmail`.