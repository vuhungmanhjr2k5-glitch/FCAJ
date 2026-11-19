---
title: "Deploy"
weight: 3
chapter: false
pre: " <b> 5.7.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

## Key for the API deployment

On first deployment the block generates a session key and copies it to the clipboard. To reuse an existing key, set the session’s `API_KEY` to the current value first.

```powershell
if ([string]::IsNullOrWhiteSpace($env:API_KEY)) {
    $env:API_KEY = [guid]::NewGuid().ToString('N')
}
Set-Clipboard -Value $env:API_KEY
```

Keep `order-system` and the existing email values in guided deployment. Paste the clipboard value at the `ApiKey` prompt. Both APIs use an application key. Keep the key and `samconfig.toml` local.

```powershell
sam validate --region ap-southeast-1
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed.' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed.' }
Set-Clipboard -Value $env:API_KEY
sam deploy --guided --region ap-southeast-1 --no-fail-on-empty-changeset
if ($LASTEXITCODE -ne 0) { throw 'Inspect the CloudFormation deployment events.' }
```

### Resolve the new deployment outputs

```powershell
$stackData = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot read order-system.' }
$outputs = @{}
foreach ($item in ($stackData | Out-String | ConvertFrom-Json).Stacks[0].Outputs) {
    $outputs[$item.OutputKey] = $item.OutputValue
}
$api = $outputs['ApiUrl']
if (-not $api) { throw 'Missing output: ApiUrl' }
$table = $outputs['TableName']
if (-not $table) { throw 'Missing output: TableName' }
$sm = $outputs['StateMachineArn']
if (-not $sm) { throw 'Missing output: StateMachineArn' }
$env:AWS_REGION = 'ap-southeast-1'
$api = $api.TrimEnd('/')
```

## Separate deployment settings from benchmark settings

`ApiKey` is the CloudFormation parameter checked against `x-api-key` by both functions. `$env:API_KEY` supplies the header value to the benchmark process in this shell. The values must match. `NoEcho` masks the parameter in certain CloudFormation views. Guided-deployment settings and local shell history still need careful handling. When redeploying without intending to rotate the key, use the current value so this session's tools remain usable.

`ApiUrl` is the base URL. Append `/async` or `/sync` once per call. Obtain the base URL from the same stack that supplies `TableName` and `StateMachineArn`. Physical names in the recorded evidence can differ from your account.

## Send one request before a measured run

```powershell
$smokeId = 'smoke-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$body = @{ orderId = $smokeId; customerEmail = 'success@simulator.amazonses.com' } | ConvertTo-Json -Compress
$response = Invoke-WebRequest -UseBasicParsing -Method Post -Uri "$api/async" -Headers @{'x-api-key' = $env:API_KEY} -ContentType 'application/json' -Body $body -ErrorAction Stop
$response | Select-Object StatusCode, Content
```

Read the status and body. The async route should return 202 with the ID just sent. Look up that ID in DynamoDB or Step Functions before measuring a group. For 403 check the session key. For 404 inspect the base URL, route and POST method. For 500 inspect the relevant function log. Keep this smoke request outside the measured population.

<!-- evidence-deployment:start -->

**Supplementary deployment:** order-api-evidence reuses order-orchestration-evidence resources. The evidence below is from November 13, 2026.

![Deployment of comparison APIs](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-api-deploy-success.png)

<!-- evidence-deployment:end -->
