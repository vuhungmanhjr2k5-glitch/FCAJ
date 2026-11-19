---
title: "Verify in the Console"
weight: 3
chapter: false
pre: " <b> 5.3.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

1. Go to **Amazon SQS** → **Queues**.
2. You will see two queues whose names start with `order-system-`.
3. Open the main queue → **Dead-letter queue** tab: the dead-letter queue is attached, with **Maximum receives = 3**.

![SQS queues](/images/5-Workshop/5.3-SQS-DLQ/queues.png)

## Compare the deployed values with the template

Open the source queue identified by `QueueUrl` and read the destination ARN in its redrive policy. Then open the queue identified by `DlqUrl` and read its own ARN. Compare the full values. Similar name suffixes can belong to different deployments in the same account.

```powershell
$raw = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the application stack.' }
$outputs = ($raw | Out-String | ConvertFrom-Json).Stacks[0].Outputs
$q = ($outputs | Where-Object OutputKey -eq 'QueueUrl').OutputValue
$dlq = ($outputs | Where-Object OutputKey -eq 'DlqUrl').OutputValue
$main = aws sqs get-queue-attributes --queue-url $q --attribute-names RedrivePolicy VisibilityTimeout --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the source queue.' }
$attrs = ($main | Out-String | ConvertFrom-Json).Attributes
$policy = $attrs.RedrivePolicy | ConvertFrom-Json
$actualArn = aws sqs get-queue-attributes --queue-url $dlq --attribute-names QueueArn --query 'Attributes.QueueArn' --region ap-southeast-1 --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the DLQ.' }
[pscustomobject]@{
    ConfiguredTarget  = $policy.deadLetterTargetArn
    ActualTarget      = $actualArn
    SameTarget        = ($policy.deadLetterTargetArn -eq $actualArn)
    MaxReceives       = $policy.maxReceiveCount
    VisibilitySeconds = $attrs.VisibilityTimeout
}
```

Expect `SameTarget` to be `True`, `MaxReceives` to be 3 and `VisibilitySeconds` to be 60. The AWS CLI returns the redrive policy as a JSON string inside the outer JSON attribute response. The two JSON parsing steps reflect these two layers.

## Resolve a mismatch before polling

If both queues exist but the policy is empty, check whether you opened the DLQ as the source. If the destination ARN belongs to another queue, use the stack **Resources** tab to match each logical ID to its physical name. For a missing queue, read the outputs again in Singapore and verify the profile. Continue to the redelivery experiment after identifying the actual destination. A row with the two ARNs and the receive limit records the configuration for this run.

### Hand the queue pair to another operator

Give the operator the stack, Region and output names. They can resolve the current URLs with their own profile and read both sets of attributes. Stored URLs can become obsolete when a stack is recreated. Resolving the outputs catches this before the first send. If `SameTarget` is false, investigate the source-to-DLQ wiring before introducing any failures.
