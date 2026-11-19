---
title: "Redeploy"
weight: 3
chapter: false
pre: " <b> 5.4.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

```powershell
sam validate --region ap-southeast-1
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed.' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed.' }
sam deploy --region ap-southeast-1 --no-fail-on-empty-changeset
if ($LASTEXITCODE -ne 0) { throw 'Inspect the CloudFormation deployment events.' }
```

![Second deploy](/images/5-Workshop/5.4-Idempotency/deploy-result.png)

![SQS trigger enabled for the Lambda function](/images/5-Workshop/5.4-Idempotency/sqs-trigger-enabled.png)

## Verify that the update reads the intended queue

After deploying to the existing stack, open **CloudFormation → Resources**. The two queues from 5.3 should remain in the stack. This update adds the table, worker and event source mapping. The deployment consumes build output, so editing `src/index.mjs` after a build requires another build before deploying that edit.

The first command below resolves `ProcessOrderFn` to its physical name. The second reads mappings for that function. Look for `Enabled`, a source ARN ending in the deployed OrdersQueue name, and a batch size of one.

```powershell
$worker = aws cloudformation describe-stack-resource --stack-name order-system --logical-resource-id ProcessOrderFn --region ap-southeast-1 --query 'StackResourceDetail.PhysicalResourceId' --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'ProcessOrderFn was not found.' }
aws lambda list-event-source-mappings --function-name $worker --region ap-southeast-1 --query 'EventSourceMappings[].{State:State,Source:EventSourceArn,Batch:BatchSize}' --output table --no-cli-pager
```

## Allow the trigger to reach its ready state

Just after deployment a mapping may still be `Creating` or `Enabling`. Wait briefly and read the same mapping again. For an empty list, compare the stack events, profile, Region and function name. If enabled with no invocations, check whether the source has messages. The next test uses a distinct order ID to exercise the whole path.

`UPDATE_COMPLETE` gives an infrastructure checkpoint. A log line for a new ID and a matching `PROCESSING` item add two application checkpoints. Keeping all three distinguishes a successful deployment from a worker reading the wrong input.

### Match the worker's table to the test target

Read `TABLE_NAME` in **Lambda → Configuration → Environment variables** and `TableName` in CloudFormation Outputs. Matching values establish that the next test will read the table targeted by the worker. If they differ, check whether the function comes from an older stack in the same Region. `OrdersTable` is the template's logical ID. The application accesses a physical table name.
