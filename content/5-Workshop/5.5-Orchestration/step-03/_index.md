---
title: "Deploy"
weight: 3
chapter: false
pre: " <b> 5.5.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

```powershell
sam validate --region ap-southeast-1
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed.' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed.' }
sam deploy --guided --region ap-southeast-1 --no-fail-on-empty-changeset
if ($LASTEXITCODE -ne 0) { throw 'Inspect the CloudFormation deployment events.' }
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

## Check the two kinds of email confirmation

Before building, select your workshop profile and enter the orchestration template directory. Compare `SenderEmail` with the identity verified in Singapore. Use an inbox someone can monitor during the lab for `AdminEmail`. Both values may refer to one inbox, although SES identity verification and SNS subscription confirmation are separate service operations.

When the stack finishes, confirm the SNS registration for the intended topic. Inspect its state with the block below, which resolves the topic ARN from the stack logical ID. A ready email subscription shows its ARN. `PendingConfirmation` means the publisher can use the topic while the email endpoint still awaits its owner's confirmation.

```powershell
$topic = aws cloudformation describe-stack-resource --stack-name order-system --logical-resource-id OrderTopic --region ap-southeast-1 --query 'StackResourceDetail.PhysicalResourceId' --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'OrderTopic was not found.' }
aws sns list-subscriptions-by-topic --topic-arn $topic --region ap-southeast-1 --query 'Subscriptions[].{Channel:Protocol,Subscription:SubscriptionArn}' --output table --no-cli-pager
```

## Verify the bindings before sending an order

Read `StateMachineArn` and `TableName` in stack outputs. Locate the physical worker in Resources and check its `STATE_MACHINE_ARN` and `TABLE_NAME` settings against those outputs. A missing parameter during deployment calls for a guided deploy from this template with both email values. Once the bindings agree, send one valid order before starting duplicate and failure tests.

### Reuse an inbox from preparation

The inbox may contain two AWS confirmation messages. SES verifies the address as an identity for the selected Region. SNS identifies a topic and offers a subscription-confirmation action. Check that the topic matches the deployed `OrderTopic`. If an update creates a different topic ARN, confirmation for an older topic does not migrate the subscription.
