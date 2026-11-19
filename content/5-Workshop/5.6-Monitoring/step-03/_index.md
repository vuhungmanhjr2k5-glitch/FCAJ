---
title: "Monitoring deployment and validation"
weight: 3
chapter: false
pre: " <b> 5.6.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Deploy using identifiers from CloudFormation

Work in the `order-system` directory after completing 5.5. Read the application outputs and resources to find queue URLs, state machine ARN, worker name and topic ARN. The following block takes each queue name from its URL because CloudWatch uses a name as the dimension. A separate monitoring stack lets you inspect and later remove the corresponding alarm resources together.

```powershell
$region = 'ap-southeast-1'
$sourceStack = 'order-system'
$monitorStack = 'order-monitoring'
$stackJson = aws cloudformation describe-stacks --stack-name $sourceStack --region $region --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the application stack.' }
$outputs = @{}
foreach ($item in ($stackJson | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$resourcesJson = aws cloudformation list-stack-resources --stack-name $sourceStack --region $region --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read stack resources.' }
$physical = @{}
foreach ($item in ($resourcesJson | Out-String | ConvertFrom-Json).StackResourceSummaries) {
    $physical[$item.LogicalResourceId] = $item.PhysicalResourceId
}
foreach ($name in 'QueueUrl','DlqUrl','StateMachineArn') {
    if (-not $outputs[$name]) { throw "Missing stack output: $name" }
}
foreach ($name in 'ProcessOrderFn','OrderTopic') {
    if (-not $physical[$name]) { throw "Missing physical resource: $name" }
}
$parameters = @(
    "SourceStack=$sourceStack"
    "SourceQueueName=$($outputs.QueueUrl.Split('/')[-1])"
    "DlqName=$($outputs.DlqUrl.Split('/')[-1])"
    "WorkerName=$($physical.ProcessOrderFn)"
    "MachineArn=$($outputs.StateMachineArn)"
    "TopicArn=$($physical.OrderTopic)"
)
aws cloudformation deploy --template-file .\monitoring.template.json --stack-name $monitorStack --region $region --parameter-overrides @parameters --no-fail-on-empty-changeset --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Monitoring deployment failed.' }
$config = [ordered]@{
    Region = $region
    QueueUrl = $outputs.QueueUrl
    DlqUrl = $outputs.DlqUrl
    LogGroup = "/aws/lambda/$($physical.ProcessOrderFn)"
    AlarmNames = @("$monitorStack-sfn-failed", "$monitorStack-lambda-errors", "$monitorStack-dlq")
}
$configPath = Join-Path $PWD.Path 'workshop-monitoring-5-6.json'
[IO.File]::WriteAllText($configPath, ($config | ConvertTo-Json -Depth 4), [Text.UTF8Encoding]::new($false))
```

The block saves a local configuration for 5.6.4 containing the Region, queue URLs, alarm names and worker log group. Regenerate it when moving accounts, changing Regions or replacing the source stack. The recorded example's physical IDs belong to its original deployment.

## Check the baseline before introducing failures

```powershell
$config = Get-Content -Raw -LiteralPath .\workshop-monitoring-5-6.json | ConvertFrom-Json
$names = @($config.AlarmNames)
aws cloudwatch describe-alarms --alarm-names @names --region $config.Region --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Enabled:ActionsEnabled}' --output table --no-cli-pager
```

Expect three alarms. Inspect their dimensions, `ActionsEnabled` and the action topic in Details. A newly created alarm can start in `INSUFFICIENT_DATA`. With `notBreaching` an idle deployment can settle to `OK` before any order produces a datapoint. Record this starting state after checking the configuration. The illustrations on this page refer to the author's separately recorded environment.

### Investigate an empty dashboard after deployment

Open one widget and inspect its Region, time range and dimensions. A correct queue with no traffic in that window can have an empty graph. A different queue name from the application output calls for redeploying the mapping against the intended source stack. Inspect alarm actions separately. Receiving datapoints and delivering email rely on different bindings, including confirmation of the SNS subscription.

## Previously recorded deployment

Alarm/dashboard resources are CREATE_COMPLETE and order-system is UPDATE_COMPLETE, establishing successful deployment.

![Original monitoring deployment](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h1.png)

The original alarms show OK and Actions enabled. This alone does not establish received email notifications.

[Original three alarm states](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h2.png)

order-workshop displays queues, message age, Step Functions and Lambda metrics.

[Original dashboard](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h3.png)

order-sfn-failed is ALARM with an update timestamp of October 31, 2026, 12:40:46 UTC. This timestamp is not used to measure latency for the new run.

[Original Step Functions alarm](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h4.png)
