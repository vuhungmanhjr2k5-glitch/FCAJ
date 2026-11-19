---
title: "Experiment procedure"
weight: 4
chapter: false
pre: " <b> 5.6.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Prerequisites are AWS CLI v2, a stable source stack, a verified SES sender and a confirmed SNS email subscription. Use `workshop-monitoring-5-6.json` generated during setup. The script below records initial alarm/DLQ state and sends three messages with fresh IDs on each run.

| Case | Input | Expected observations to check |
|---|---|---|
| Success | Valid SES simulator recipient | SUCCEEDED execution and DONE order |
| Email failure | not-an-email | Step Functions failure/retries, FAILED execution and sfn-failed alarm |
| Malformed JSON | not-json-mon56-... string | Lambda JSON parsing failure, SQS retries and DLQ arrival |

With the section 5.5 code, Lambda returns after starting the workflow. A later SES failure need not increase Lambda Errors. Malformed JSON independently tests Lambda and the DLQ. An SQS MessageId proves acceptance, not processing completion.

```powershell
& {
  param([string]$ConfigPath = (Join-Path $PWD.Path 'workshop-monitoring-5-6.json'))
  $ErrorActionPreference = 'Stop'
  function Read-Aws([string[]]$Arguments) {
    $raw = & aws @Arguments --region $c.Region --output json --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw ('AWS CLI failed: '+($Arguments -join ' ')) }
    return (($raw -join [Environment]::NewLine) | ConvertFrom-Json)
  }
  try {
    $c = Get-Content -LiteralPath $ConfigPath -Raw | ConvertFrom-Json
    $id = 'mon56-'+(Get-Date -Format 'yyyyMMdd-HHmmss')+'-'+[guid]::NewGuid().ToString('N').Substring(0,6)
    $dir = Join-Path (Split-Path -Parent $ConfigPath) $id
    $null = New-Item -ItemType Directory -Path $dir
    $utf8 = New-Object System.Text.UTF8Encoding($false)
    $before = Read-Aws -Arguments (@('cloudwatch','describe-alarms','--alarm-names') + @($c.AlarmNames))
    [IO.File]::WriteAllText((Join-Path $dir 'alarms-before.json'),($before | ConvertTo-Json -Depth 30),$utf8)
    $queueBefore = Read-Aws -Arguments @('sqs','get-queue-attributes','--queue-url',$c.DlqUrl,'--attribute-names','ApproximateNumberOfMessages','ApproximateNumberOfMessagesNotVisible')
    [IO.File]::WriteAllText((Join-Path $dir 'dlq-before.json'),($queueBefore | ConvertTo-Json -Depth 10),$utf8)
    Write-Host '=== ALARMS BEFORE TEST ===' -ForegroundColor Cyan
    $before.MetricAlarms | Select-Object AlarmName,StateValue | Format-Table -AutoSize
    $good = @{orderId=($id+'-ok');customerEmail='success@simulator.amazonses.com';acceptedAt=[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()}
    $bad = @{orderId=($id+'-ses-fail');customerEmail='not-an-email';acceptedAt=[DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds()}
    $cases = @(
      @{Name='success';Body=($good | ConvertTo-Json -Compress)},
      @{Name='ses-failure';Body=($bad | ConvertTo-Json -Compress)},
      @{Name='malformed-json';Body=('not-json-'+$id)}
    )
    $results = @()
    foreach ($case in $cases) {
      $file = Join-Path $dir ($case.Name+'.txt')
      [IO.File]::WriteAllText($file,$case.Body,$utf8)
      $sent = Read-Aws -Arguments @('sqs','send-message','--queue-url',$c.QueueUrl,'--message-body',('file://'+$file))
      $results += [pscustomobject]@{Case=$case.Name;MessageId=$sent.MessageId;SentAtUtc=[DateTimeOffset]::UtcNow.ToString('o')}
      [IO.File]::WriteAllText((Join-Path $dir 'sent.json'),($results | ConvertTo-Json -Depth 10),$utf8)
    }
    $run = @{RunId=$id;GoodOrderId=$good.orderId;FailedOrderId=$bad.orderId;MalformedBody=('not-json-'+$id);EvidenceDirectory=$dir}
    [IO.File]::WriteAllText((Join-Path $dir 'run.json'),($run | ConvertTo-Json -Depth 10),$utf8)
    Write-Host '=== TEST MESSAGES ACCEPTED BY SQS ===' -ForegroundColor Green
    $results | Format-Table -AutoSize
    $run | Format-List
    Write-Host 'Da gui 3 tin; chua ket luan xu ly thanh cong. Theo doi Step Functions, Lambda, DLQ va CloudWatch trong 5-10 phut.'

    Write-Host 'DLQ co san tin cu thi ALARM/email cu khong chung minh do tin vua gui.'
  } catch {
    Write-Host ('LOI: '+$_.Exception.Message) -ForegroundColor Red
    throw
  }
}
```

Observe for an initial 5–10 minutes and allow additional time for retries and metric publication. Find executions by orderId; correlate Lambda logs, the DLQ queue/body, alarm Details/History and matching email subjects. Do not use set-alarm-state as evidence of an actual application failure.

## Predict the three signals before you send

The valid simulator order can start a successful workflow. An invalid address is still valid JSON, so the worker can start the workflow before SES rejects the recipient. The third message stops at parsing. SQS can accept all three, but only the first two can start executions with a business ID. Search the Lambda log for the malformed text case.

One run keeps its payloads and send receipts together in a uniquely named directory. Inspect `alarms-before.json` to learn which alarms were already in `ALARM`. Read the visible and in-flight counts in `dlq-before.json`. Existing backlog is part of the baseline and can be left in place while interpreting the test.

## Build a timeline after submission

| Milestone | Source | Use |
|---|---|---|
| Send receipt | `sent.json` and message ID | Anchor the client's observation of SQS acceptance |
| Application failure | Execution history or parsing log | Associate the first failure with its case |
| Alarm transition | CloudWatch alarm history | Identify a change that actually occurred |
| Mailbox arrival | SNS email naming the alarm | Record arrival if measuring notification delivery |

Mark an unobserved milestone as pending and retain the others. Running the script again creates another three messages and a new set of IDs. Read the current results first, especially when a persistent `ALARM` state will not transition again for each message.

### Reopen the observations without sending more messages

Use the `EvidenceDirectory` printed by the run. The following block reads local data only. Match the rows in `sent.json` to the IDs in `run.json`. A run interrupted after its first accepted send may contain just one receipt. Include only cases with a recorded submission in that comparison.

```powershell
$dir = Read-Host 'EvidenceDirectory'
Get-Content -LiteralPath (Join-Path $dir 'sent.json') -Raw | ConvertFrom-Json | Format-Table Case, MessageId, SentAtUtc
$runFile = Join-Path $dir 'run.json'
if (Test-Path -LiteralPath $runFile) {
    Get-Content -LiteralPath $runFile -Raw | ConvertFrom-Json | Format-List
}
```

`SentAtUtc` was recorded after the send call returned and identifies a client-side observation. Keep it separate from the start of a Lambda invocation when building a detailed timeline. If an interruption prevented `run.json` from being written, retain the receipts and bodies already saved. Check their identifiers before repeating the experiment because some work may still finish on AWS.

### Read worker logs for the submitted run

Run from the application directory. Keep the run time beside the log window when distinguishing invocations from different trials.

```powershell
$config = Get-Content -Raw -LiteralPath .\workshop-monitoring-5-6.json | ConvertFrom-Json
aws logs tail $config.LogGroup --since 15m --region $config.Region --format short --no-cli-pager
```
