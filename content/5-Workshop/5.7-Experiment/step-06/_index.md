---
title: "Experiment 2: duplicate orders"
weight: 6
chapter: false
pre: " <b> 5.7.6 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Identical messages for **dup-proof-20261113-220937-a7468d** were submitted to SQS **five times**. Results: execution **SUCCEEDED**, **one** successful SES task in that execution, DynamoDB **DONE**, and **emailCount = 1**.

![Duplicate-test validation](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-validation.png)

Logs show duplicates being skipped. Two workflow-started log lines do not establish two executions: Standard StartExecution with the same name and input can return an existing running execution.

[Logs showing duplicate orders skipped](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-skip-logs.png)

The inbox received a confirmation with the matching order ID. These observations support duplicate handling in this test, not guaranteed exactly-once email delivery under all failures.

[Received confirmation email](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/duplicate-confirmation-email.png)

## Replay one purchase with a stable identity

The recorded trial submitted **one** body directly to SQS five times. This exercises the downstream worker and workflow. The intake API is outside this particular test. For a repeat, choose a verified test inbox to observe email delivery or the simulator to observe only SES task success. Keep the ID and business payload identical across the sends. A new random ID each time would be a different experiment.

```powershell
$q = aws cloudformation describe-stacks --stack-name order-system --query "Stacks[0].Outputs[?OutputKey=='QueueUrl'].OutputValue | [0]" --region ap-southeast-1 --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $q -eq 'None') { throw 'QueueUrl unavailable.' }
$dupId = 'dup-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$email = Read-Host 'Verified test recipient (Enter = SES simulator)'
if ([string]::IsNullOrWhiteSpace($email)) { $email = 'success@simulator.amazonses.com' }
$body = @{ orderId = $dupId; customerEmail = $email } | ConvertTo-Json -Compress
$file = Join-Path $PWD.Path "$dupId.json"
[IO.File]::WriteAllText($file, $body, [Text.UTF8Encoding]::new($false))
1..5 | ForEach-Object {
    aws sqs send-message --queue-url $q --message-body "file://$file" --region ap-southeast-1 --query MessageId --output text --no-cli-pager
    if ($LASTEXITCODE -ne 0) { throw "Send $_ failed." }
}
$dupId
```

Each accepted send returns a message ID. Five submissions describe one purchase. Locate the execution named for `$dupId` and count successful SES tasks in its history. Then inspect `status` and `emailCount` under the same table key. For a real inbox, find the subject containing the fresh ID. Reading results requires no sixth submission.

## Investigate counts that disagree

For two “started” log lines, count distinct execution ARNs first. For one execution with multiple successful SES tasks, inspect state history, timing and retries. For a single item with a counter greater than one, verify the template version and run ID. Each observation narrows the next investigation. Report actual mailbox count only when you have inspected that inbox.
