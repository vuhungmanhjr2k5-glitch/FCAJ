---
title: "Experiment 1: latency and load"
weight: 5
chapter: false
pre: " <b> 5.7.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

On **November 13, 2026**, two APIs in **order-api-evidence** were tested using the DynamoDB table and workflow in **order-orchestration-evidence**. Run ID: **cmp-20261113160505330-eac6bb**.

Each mode used **20 measured requests**, concurrency **1**, a **1,500 ms pause after each response**, and two excluded warm-up requests. Synchronous ran first, then asynchronous. Recipient: success@simulator.amazonses.com. The actual tool was workshop-benchmark-tools/compare.cjs; illustrative tools above may use a different output format.

### Synchronous results

![Synchronous results for 20 orders](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-sync-20.png)

### Asynchronous results

[Asynchronous results for 20 orders](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-async-20.png)

### Comparison

| Metric | Synchronous | Asynchronous |
|---|---:|---:|
| Measured requests / concurrency | 20 / 1 | 20 / 1 |
| Pause after response (ms) | 1500 | 1500 |
| Successful HTTP requests | 20/20 | 20/20 |
| HTTP / network errors | 0 / 0 | 0 / 0 |
| Average API latency (ms) | 229.55 | 109.28 |
| P95 API latency (ms) | 343.33 | 110.62 |
| Records found / DONE | 20 / 20 | 20 / 20 |
| FAILED / other / not observed | 0 / 0 / 0 | 0 / 0 / 0 |
| Completion timestamp samples | 20 | 20 |
| Average approximate completion (ms) | 122.75 | 445.55 |
| P95 approximate completion (ms) | 138 | 516 |

[Comparison with matching test settings](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-summary.png)

**Observation:** the asynchronous API responded faster in this run, while approximate completion took longer. HTTP 202 confirms acceptance; DONE is checked separately in DynamoDB. Client API latency excludes the pacing pause. Approximate completion uses application createdAt and doneAt, with different measurement boundaries; it is neither inbox delivery time nor exact Step Functions duration.

This is one small paced test per mode, not a maximum-throughput measurement. Equal pauses after responses do not ensure equal arrival rates. Downloads/workshop-api-evidence.json contains an API key and must not be published.

## Repeat the measurement on your stack

Work in `order-system` using the session that set `$api`, `$table` and the key in 5.7.3. This block uses the two endpoints belonging to the workshop stack. Each path uses two warm-ups with separate IDs. The example client in 5.7.4 supports a 1,500 ms pause after a response and before the next request. Your output is a new run. The original table identifies its own benchmark tool and date.

```powershell
$runId = 'cmp-' + [guid]::NewGuid().ToString('N').Substring(0,10)
$env:CUSTOMER_EMAIL = 'success@simulator.amazonses.com'
foreach ($mode in 'sync','async') {
    $prefix = "$runId-$mode"
    node .\tools\loadtest.mjs "$api/$mode" "$prefix-warm" 2 1 1500
    if ($LASTEXITCODE -ne 0) { throw "Warm-up failed for $mode." }
    Start-Sleep -Milliseconds 1500
    node .\tools\loadtest.mjs "$api/$mode" "$prefix-measured" 20 1 1500
    if ($LASTEXITCODE -ne 0) { throw "Check the run for $mode before continuing." }
    Get-Date -Format o
    node .\tools\report.mjs $table "$prefix-measured" 20
    if ($LASTEXITCODE -ne 0) { throw 'Report failed.' }
}
```

If the async report still shows `RECEIVED` or some IDs are absent, allow more processing time and rerun only the report with the same prefix. Repeating the whole block creates new orders. For 20 timed responses nearest-rank P95 is the nineteenth sorted value. A network exception can reduce the timed sample count below 20.

```powershell
Get-Date -Format o
node .\tools\report.mjs $table "$runId-async-measured" 20
```

## Read both differences in the recorded table

Mean API latency went from 229.55 to 109.28 ms, a difference of 120.27 ms in this run. Approximate mean completion time went from 122.75 to 445.55 ms, a difference of 322.80 ms in the other direction. The async API responded while downstream work was still underway. Both groups eventually contained 20 completed items. A different concurrency or SES throttling regime calls for a separate table with its own conditions and observation time.
