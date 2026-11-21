---
title: "Resource cleanup"
weight: 8
chapter: false
pre: " <b> 5.8 </b> "
---

<!-- workshop-merged-58:v1 -->

## Objective

Delete all the resources you created so that no charges are incurred,
especially the two public APIs from section 5.7.

## Before deletion

Save the 5.7 results and check the profile, Region and stack names. If you deployed the separate monitoring stack in 5.6.3, remove it first to retire alarms pointing at resources about to be deleted. Run the following block only for your dedicated `order-monitoring` lab stack.

```powershell
$monitorStack = 'order-monitoring'
aws cloudformation describe-stacks --stack-name $monitorStack --region ap-southeast-1 --query 'Stacks[0].{Stack:StackName,Status:StackStatus}' --output table --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Check the monitoring stack name and Region.' }
if ((Read-Host "Type $monitorStack to remove this lab monitoring stack") -ceq $monitorStack) {
    aws cloudformation delete-stack --stack-name $monitorStack --region ap-southeast-1
    if ($LASTEXITCODE -ne 0) { throw 'Deletion request failed.' }
    aws cloudformation wait stack-delete-complete --stack-name $monitorStack --region ap-southeast-1
    if ($LASTEXITCODE -ne 0) { throw 'Inspect stack events before continuing.' }
}
```

## 1. Delete the main stack

```powershell
sam delete --stack-name order-system --region ap-southeast-1
```

Answer `y` to the confirmation prompts. This command deletes the
CloudFormation stack together with SQS, the DLQ, DynamoDB, Lambda, Step
Functions, SNS, other resources in this stack, and the HTTP API.

Check that the latest deployment of the stack is `DELETE_COMPLETE`. CloudFormation can retain earlier stack history.

```powershell
aws cloudformation list-stacks --region ap-southeast-1 --query "StackSummaries[?StackName=='order-system'].[StackName,StackStatus,CreationTime]" --output table --no-cli-pager
```

## 2. Delete the Lambda log groups

Lambda creates its log groups automatically, so they are not deleted with the
stack:

```powershell
$groups = aws logs describe-log-groups --region ap-southeast-1 --log-group-name-prefix '/aws/lambda/order-system-' --query 'logGroups[].logGroupName' --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot list log groups.' }
foreach ($name in ($groups | Out-String | ConvertFrom-Json)) {
    if ((Read-Host "Type DELETE to remove the lab log group $name") -ceq 'DELETE') {
        aws logs delete-log-group --region ap-southeast-1 --log-group-name $name
        if ($LASTEXITCODE -ne 0) { throw "Could not delete $name." }
    }
}
```

## 3. Delete the SAM managed stack (optional)

Only remove SAM resources dedicated to this lab. SAM also creates the `aws-sam-cli-managed-default` stack along with an S3
bucket that holds the deployment code. The cost is very small, but you can
clean it up: go to **S3**, open the bucket whose name starts with
`aws-sam-cli-managed-default`, choose **Empty**, then go to **CloudFormation**
and delete the `aws-sam-cli-managed-default` stack.

## 4. Delete the verified email in SES (optional)

Go to **Amazon SES** → **Identities** → select the email → **Delete**.

## 5. Revoke access

Do this step **last**, because once the key is deleted the CLI can no longer
be used:

1. Go to **IAM** → **Users** → select the user → **Security credentials** → **Deactivate** and then **Delete** the access key.
2. Detach the `OrderWorkshopDeployPolicy` policy from the user, then delete the policy if it is no longer needed.
3. Remove the profile from your machine: open the `%USERPROFILE%\.aws\credentials` and `config` files and delete the `order-workshop` section.
4. Remove the environment variables in PowerShell:

```powershell
Remove-Item Env:API_KEY, Env:CUSTOMER_EMAIL -ErrorAction SilentlyContinue
```

## 6. Final check

1. Go to the Console for **Lambda**, **SQS**, **DynamoDB**, **Step Functions**, **API Gateway**, and **CloudWatch** (Alarms, Dashboards): no `order-system` resources should remain.
2. Go to **Billing and Cost Management** → **Cost Explorer** and check again after 1 to 2 days for new charges from resources you removed.
3. Keep the budget in **AWS Budgets** if you are still using this account.


## 7. Cleanup checklist

| Task | Done |
|---|---|
| Delete the `order-system` stack | ☑ |
| Delete the Lambda log groups | ☑ |
| Delete the SAM stack and bucket (optional) | ☑ |
| Delete the SES identity (optional) | ☑ |
| Delete the IAM access key and policy | ☑ |
| Remove the profile and environment variables on the machine | ☑ |
| Check Cost Explorer after 1 to 2 days | ☑ |
