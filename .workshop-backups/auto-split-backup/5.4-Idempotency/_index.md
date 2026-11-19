---
title: "Preventing Duplicate Processing"
weight: 4
chapter: false
pre: " <b> 5.4 </b> "
---
## Objective

An SQS queue guarantees **at-least-once** delivery, which means the same order
may occasionally be delivered twice. If this is not handled, customers will
receive duplicate emails. We use **DynamoDB** as a "ledger": each `orderId` is
processed only once. This technique is called **idempotency**.

#### 1. Update template.yaml

Add a DynamoDB table and a Lambda function to `template.yaml`. Keep `OrdersDLQ`
and `OrdersQueue` as they are, and replace the rest as follows:

```yaml
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
```

> The queue's `VisibilityTimeout` (60 seconds) is several times larger than the
> Lambda `Timeout` (10 seconds). This is the recommended configuration when
> Lambda reads from SQS.

#### 2. Write the Lambda code

Create the file `src/index.mjs`:

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async (event) => {
  for (const record of event.Records) {
    const order = JSON.parse(record.body); // invalid JSON throws an error -> retry -> DLQ

    try {
      await db.send(new PutCommand({
        TableName: process.env.TABLE_NAME,
        Item: {
          orderId: order.orderId,
          status: "PROCESSING",
          createdAt: Date.now(),
        },
        // Only write if the orderId does not exist yet
        ConditionExpression: "attribute_not_exists(orderId)",
      }));
    } catch (e) {
      if (e.name === "ConditionalCheckFailedException") {
        console.log("Duplicate order, skipping:", order.orderId);
        continue;
      }
      throw e;
    }

    console.log("Processed:", order.orderId);
  }
};
```

The key point is the `ConditionExpression`: DynamoDB only allows the write if
the `orderId` does not already exist, so the second write is rejected and the
Lambda skips that order.

> In section 5.5 we will add an order status (`PROCESSING` → `DONE`) to handle
> the case where the order was recorded in the ledger but the email-sending
> step failed.

#### 3. Redeploy

```powershell
sam build
sam deploy
```

![Second deploy](/images/5-Workshop/5.4-Idempotency/deploy-result.png)

![SQS trigger enabled for the Lambda function](/images/5-Workshop/5.4-Idempotency/sqs-trigger-enabled.png)

#### 4. Test

Get the variables you need:

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue" --output text
$dlq = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='DlqUrl'].OutputValue" --output text
$table = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='TableName'].OutputValue" --output text
```

Create the order message file:

```powershell
'{"orderId":"A001"}' | Out-File -Encoding ascii msg1.json
```

**Case 1: valid order**

```powershell
aws sqs send-message --queue-url $q --message-body file://msg1.json
aws dynamodb scan --table-name $table
```

The DynamoDB table has one row `A001` with `status = PROCESSING`.

![Data in DynamoDB](/images/5-Workshop/5.4-Idempotency/dynamodbb-item.png)

**Case 2: sending the same order twice**

```powershell
aws sqs send-message --queue-url $q --message-body file://msg1.json
sam logs --stack-name order-system --name ProcessOrderFn --tail
```

The log shows the line `Duplicate order, skipping: A001`. Press `Ctrl+C` to exit log tailing mode.
The DynamoDB table still has only one `A001` row.

![Lambda skips the resubmitted order A001](/images/5-Workshop/5.4-Idempotency/duplicate-logg.png)

![DynamoDB retains one A001 record after resubmission](/images/5-Workshop/5.4-Idempotency/duplicate-item.png)



**Case 3: a faulty message goes to the DLQ**

```powershell
aws sqs send-message --queue-url $q --message-body "not-json"
```

The Lambda fails at `JSON.parse`, and SQS retries every 60 seconds. After about
3 to 4 minutes, the message appears in the DLQ (Console → DLQ → **Poll for messages**).

![Invalid message found in the DLQ](/images/5-Workshop/5.4-Idempotency/dlqq-message.png)

*Retest evidence from the `order-idempotency-evidence` stack:
the `khong-phai-json` message is present in the dead-letter queue.*

## Verify the result

| Case | Expected result |
|---|---|
| Valid order | One row in DynamoDB |
| Duplicate send | The log reports a duplicate, no new row is added |
| Message that is not JSON | After a few retries, the message ends up in the DLQ |

## Troubleshooting

- **Lambda does not run**: go to Console → Lambda → the `ProcessOrderFn` function → **Configuration → Triggers** tab, and check that the SQS trigger is *Enabled*.
- **DynamoDB permission error**: check the log, and make sure the `Policies` section of the template uses the correct `!Ref OrdersTable`.
- **`sam logs` shows nothing**: wait a few seconds and send the message again, as logs have a small delay.