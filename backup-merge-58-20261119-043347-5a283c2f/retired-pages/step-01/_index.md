---
title: "Delete the main stack"
weight: 1
chapter: false
pre: " <b> 5.8.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

```powershell
sam delete --stack-name order-system --region ap-southeast-1
```

Answer `y` to the confirmation prompts. This command deletes the
CloudFormation stack together with SQS, the DLQ, DynamoDB, Lambda, Step
Functions, SNS, the CloudWatch alarms, the dashboard, and the HTTP API.

Check that the stack is gone:

```powershell
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE DELETE_FAILED --query "StackSummaries[].StackName" --output text
```

![Delete stack result](/images/5-Workshop/5.8-Cleanup/delete-stack.png)
