---
title: "Chuẩn bị công cụ đo"
weight: 4
chapter: false
pre: " <b> 5.7.4 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

## Chuẩn bị thư mục đo và thư viện đọc bảng

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

## Gửi yêu cầu và đo thời gian phía máy khách

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

## Đọc trạng thái đơn sau khi gửi

```powershell
$content = @'
// Cach dung: node report.mjs <ten-bang> <tien-to> <so-don-da-gui>
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

console.log(`Tien to: ${prefix} | gui: ${n} | tim thay: ${items.length} | chua thay tai luc doc: ${Number(n) - items.length}`);
console.log("Trang thai:", status);
console.log("Email du (gui trung):", extra);
console.log(`Thoi gian hoan tat (ms): avg=${avg} p95=${p95} (so don: ${d.length})`);
if (items.length <= 10)
  console.table(items.map(i => ({ orderId: i.orderId, status: i.status, emailCount: i.emailCount ?? "N/A" })));
'@
$path = Join-Path $PWD.Path 'tools/report.mjs'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Hiểu hai tệp công cụ theo đầu vào và đầu ra

Công cụ gửi tải nhận URL một endpoint, tiền tố, số yêu cầu và số worker đồng thời. Mỗi worker đợi phản hồi, đọc xong body rồi ghi một thời gian tính trên máy khách. Khoảng nghỉ giữa hai yêu cầu, nếu bật, nằm ngoài thời gian API. Một phản hồi HTTP lỗi vẫn có thời gian phản hồi để phân tích, còn lỗi kết nối trước khi có HTTP thuộc nhóm lỗi mạng. Ghi cả số mẫu có thời gian để biết P95 đại diện cho bao nhiêu phản hồi.

Công cụ báo cáo đọc dữ liệu sau phép gửi. Với bảng nhỏ của bài học, Scan phân trang rồi lọc theo tiền tố là cách đơn giản để đối chiếu. Bộ lọc không làm giảm số item DynamoDB phải đọc trước khi lọc. Chạy báo cáo một lần sau khi hệ thống có thời gian xử lý, sau đó kiểm tra lại nếu vẫn còn `RECEIVED`. Ghi thời điểm chốt ở cả hai lần. Tránh chạy vòng lặp Scan liên tục trong lúc đang đo.

## Quy ước mã lượt và cách đọc N/A

Tạo tiền tố gồm ngày giờ, hậu tố ngẫu nhiên và tên phương án. Hai nhóm `...-sync` và `...-async` không dùng chung mã đơn. Hai yêu cầu warm-up dùng tiền tố riêng, để phép Scan của nhóm đo không đếm chúng. Khi `doneAt` hoặc `createdAt` còn thiếu, công cụ loại item đó khỏi phép đo thời gian hoàn tất và báo số mẫu còn lại. Nếu chưa có `emailCount`, ghi N/A cho chỉ số email dư. Ô thiếu dữ liệu không mang cùng nghĩa với một kết quả đo bằng 0.

Kiểm tra cú pháp các tệp sau khi lưu. Thao tác này chỉ đọc mã, nên có thể thực hiện trước khi truyền URL và khóa.

```powershell
node --check .\tools\loadtest.mjs
node --check .\tools\report.mjs
```
