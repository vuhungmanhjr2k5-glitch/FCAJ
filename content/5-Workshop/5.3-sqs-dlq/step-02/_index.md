---
title: "Deploy"
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
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

## Follow deployment through to the queue URLs

SAM reads the application template and prepares local build output under `.aws-sam`. Deployment hands the change set to CloudFormation. At this stage expect changes for two queues. Keep the `order-system` name and Singapore Region for the updates in sections 5.4 and 5.5. Creating a stack for each chapter could attach the later worker to a different queue from the one you are testing.

After deployment, read the outcome with the same profile. Use the URLs returned in your own account. Their final path components contain physical resource names and can differ from the examples in the article.

```powershell
aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs}' --output json --no-cli-pager
```

## Interpret the stack status

| Status | Interpretation | Next action |
|---|---|---|
| `CREATE_IN_PROGRESS` | CloudFormation is applying the change set | Wait for the stack completion event |
| `CREATE_COMPLETE` | Provisioning finished | Read both outputs and inspect the redrive policy |
| `ROLLBACK_COMPLETE` after initial creation | CloudFormation rolled back the failed creation | In **Events**, find the first resource error and address its cause before recreating |

Redeploying an unchanged template can report an empty change set. Read the existing stack status and outputs. If the event list only seems to show resources marked “creation cancelled,” read back to the earlier concrete failure, such as a denied queue creation.
