---
title: "Prepare the measurement tools"
weight: 4
chapter: false
pre: " <b> 5.7.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

In PowerShell, start in the `order-system` application directory and run the blocks in order. File writers use UTF-8 without a BOM. Save local customizations before replacing a file of the same name.

## Prepare the tools directory and table client

```powershell
New-Item -ItemType Directory -Path .\tools -Force -ErrorAction Stop | Out-Null
```

```powershell
Push-Location -LiteralPath .\tools -ErrorAction Stop
try {
    if (-not (Test-Path -LiteralPath .\package.json)) {
        npm.cmd init -y
        if ($LASTEXITCODE -ne 0) { throw 'npm init failed.' }
    }
    npm.cmd install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
    if ($LASTEXITCODE -ne 0) { throw 'Dependency installation failed.' }
} finally {
    Pop-Location
}
```

## Submit requests and time the client response

```powershell
$content = @'
const [target, prefix, countArg, workersArg, pauseArg = "0"] = process.argv.slice(2);
const count = Number(countArg);
const concurrency = Number(workersArg);
const pauseMs = Number(pauseArg);
const key = process.env.API_KEY;
const customerEmail = process.env.CUSTOMER_EMAIL || "success@simulator.amazonses.com";

if (!target || !key || !/^[A-Za-z0-9_-]{1,54}$/.test(prefix ?? "") ||
    !Number.isSafeInteger(count) || count < 1 ||
    !Number.isInteger(concurrency) || concurrency < 1 || concurrency > count ||
    !Number.isInteger(pauseMs) || pauseMs < 0) {
  console.error("Usage: loadtest.mjs <workshop-url> <prefix> <count> <concurrency> [pause-ms]. Set API_KEY.");
  process.exit(1);
}

const latencies = [];
let accepted = 0, httpErrors = 0, networkErrors = 0;
let next = 1;

async function worker() {
  while (next <= count) {
    const orderId = `${prefix}-${next++}`;
    const body = JSON.stringify({ orderId, customerEmail });
    const start = performance.now();
    try {
      const response = await fetch(target, {
        method: "POST",
        headers: { "content-type": "application/json", "x-api-key": key },
        body,
        signal: AbortSignal.timeout(30000),
      });
      await response.text();
      latencies.push(performance.now() - start);
      if (response.ok) accepted++;
      else httpErrors++;
    } catch {
      networkErrors++;
    }
    if (pauseMs && next <= count) await new Promise(resolve => setTimeout(resolve, pauseMs));
  }
}

await Promise.all(Array.from({ length: concurrency }, worker));
latencies.sort((a, b) => a - b);
const samples = latencies.length;
const mean = samples ? latencies.reduce((sum, ms) => sum + ms, 0) / samples : null;
const p95 = samples ? latencies[Math.ceil(samples * 0.95) - 1] : null;
console.table([{ prefix, count, concurrency, pauseMs, accepted, httpErrors, networkErrors,
  samples, avgMs: mean?.toFixed(2) ?? "N/A", p95Ms: p95?.toFixed(2) ?? "N/A" }]);
if (httpErrors || networkErrors) process.exitCode = 2;
'@
$path = Join-Path $PWD.Path 'tools/loadtest.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Read order outcomes after submission

```powershell
$content = @'
// Usage: node report.mjs <table-name> <prefix> <orders-sent>
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, ScanCommand } from "@aws-sdk/lib-dynamodb";

const [table, prefix, n] = process.argv.slice(2);
const db = DynamoDBDocumentClient.from(new DynamoDBClient({}));

const items = [];
let key;
do {
  const r = await db.send(new ScanCommand({
    TableName: table,
    ConsistentRead: true,
    FilterExpression: "begins_with(orderId, :p)",
    ExpressionAttributeValues: { ":p": `${prefix}-` },
    ExclusiveStartKey: key,
  }));
  items.push(...(r.Items ?? []));
  key = r.LastEvaluatedKey;
} while (key);

const status = {};
for (const i of items) status[i.status] = (status[i.status] ?? 0) + 1;
const extra = items.length && items.every(i => Number.isFinite(i.emailCount)) ? items.reduce((s, i) => s + Math.max(0, i.emailCount - 1), 0) : "N/A";
const d = items.filter(i => i.doneAt && i.createdAt)
  .map(i => Date.parse(i.doneAt) - i.createdAt).sort((a, b) => a - b);
const avg = d.length ? Math.round(d.reduce((s, x) => s + x, 0) / d.length) : null;
const p95 = d.length ? d[Math.min(d.length - 1, Math.ceil(d.length * 0.95) - 1)] : null;

console.log(`Prefix: ${prefix} | sent: ${n} | found: ${items.length} | not seen at read time: ${Number(n) - items.length}`);
console.log("Status:", status);
console.log("Extra emails (duplicates):", extra);
console.log(`Completion time (ms): avg=${avg} p95=${p95} (orders: ${d.length})`);
if (items.length <= 10)
  console.table(items.map(i => ({ orderId: i.orderId, status: i.status, emailCount: i.emailCount ?? "N/A" })));
'@
$path = Join-Path $PWD.Path 'tools/report.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Read the two tools by input and output

The load client accepts an endpoint URL, prefix, request count and worker concurrency. Each worker waits for a response, reads its body and records a client duration. Any configured pause between requests lies outside the latency sample. An HTTP error can have a measured response time. A network exception may produce no HTTP response at all. Report the timed sample count beside P95.

The report reads the table after submission. For a small lab table a paginated Scan with a prefix filter is convenient. Filtering does not reduce how many items DynamoDB must read before applying the filter. Run the report after allowing time for processing and check again if `RECEIVED` remains. Record both observation times instead of repeatedly scanning during the measurement.

## Name populations and interpret missing values

Use a timestamp, short unique suffix and the mode in the prefix. Distinct `...-sync` and `...-async` groups keep business IDs apart. Put warm-up requests under separate prefixes so the report excludes them. A missing `doneAt` or `createdAt` excludes the item from completion-time samples. A missing `emailCount` makes the extra-email measure unavailable. A missing value and a measured zero express different results.

Check both saved files locally before supplying a URL or key.

```powershell
node --check .\tools\loadtest.mjs
node --check .\tools\report.mjs
```
