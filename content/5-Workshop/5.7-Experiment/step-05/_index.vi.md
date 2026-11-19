---
title: "Thí nghiệm 1: độ trễ và tải"
weight: 5
chapter: false
pre: " <b> 5.7.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Ngày **13/11/2026**, kiểm thử hai API của stack **order-api-evidence**, dùng chung bảng DynamoDB và quy trình **order-orchestration-evidence**. Mã nhóm: **cmp-20261113160505330-eac6bb**.

Mỗi phương án đo **20 yêu cầu**, đồng thời **1**, nghỉ **1.500 ms sau mỗi phản hồi**, loại hai yêu cầu khởi động. Đồng bộ chạy trước, bất đồng bộ chạy sau. Người nhận: success@simulator.amazonses.com. Công cụ thực tế: workshop-benchmark-tools/compare.cjs; các công cụ minh họa phía trên có thể khác định dạng đầu ra.

### Kết quả đồng bộ

![Kết quả đồng bộ với 20 đơn](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-sync-20.png)

### Kết quả bất đồng bộ

[Kết quả bất đồng bộ với 20 đơn](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-async-20.png)

### So sánh

| Chỉ số | Đồng bộ | Bất đồng bộ |
|---|---:|---:|
| Yêu cầu đo / đồng thời | 20 / 1 | 20 / 1 |
| Nghỉ sau phản hồi (ms) | 1500 | 1500 |
| HTTP thành công | 20/20 | 20/20 |
| Lỗi HTTP / mạng | 0 / 0 | 0 / 0 |
| API trung bình (ms) | 229,55 | 109,28 |
| API P95 (ms) | 343,33 | 110,62 |
| Bản ghi tìm thấy / DONE | 20 / 20 | 20 / 20 |
| FAILED / khác / chưa quan sát thấy | 0 / 0 / 0 | 0 / 0 / 0 |
| Mẫu thời gian hoàn tất | 20 | 20 |
| Hoàn tất xấp xỉ trung bình (ms) | 122,75 | 445,55 |
| Hoàn tất xấp xỉ P95 (ms) | 138 | 516 |

[Bảng so sánh cùng điều kiện cấu hình](/images/5-Workshop/5.7-Experiments/evidence-20261113-234400-f48a26/comparison-summary.png)

**Nhận xét:** API bất đồng bộ phản hồi nhanh hơn trong lần đo này; thời gian hoàn tất xấp xỉ dài hơn. HTTP 202 xác nhận tiếp nhận; DONE được kiểm tra riêng trong DynamoDB. Độ trễ API đo tại máy khách, không tính khoảng nghỉ. Thời gian hoàn tất dùng createdAt và doneAt của ứng dụng, có mốc đo khác API; không phải thời gian email đến hộp thư hoặc duration chính xác của Step Functions.

Đây là một lần đo với tải nhỏ cho mỗi phương án, chưa chứng minh thông lượng cực đại. Cùng khoảng nghỉ sau phản hồi không đồng nghĩa cùng tốc độ đến. File Downloads/workshop-api-evidence.json chứa API key, không đưa lên website hoặc kho mã.

## Lặp lại điều kiện đo trên stack của bạn

Chạy tại thư mục `order-system` trong phiên đã thiết lập `$api`, `$table` và khóa ở 5.7.3. Khối này chỉ dùng hai endpoint của stack workshop. Hai lượt khởi động cho mỗi nhánh dùng một nhóm mã riêng. Công cụ minh họa ở 5.7.4 cho phép đặt khoảng nghỉ 1.500 ms giữa các phản hồi và yêu cầu tiếp theo. Các kết quả bạn chạy là một lượt đo mới; số trong bảng gốc thuộc công cụ và ngày đo đã ghi ở trên.

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

Nếu báo cáo bất đồng bộ vẫn còn item `RECEIVED` hoặc chưa thấy đủ mã, để workflow chạy tiếp rồi chỉ đọc lại báo cáo, giữ nguyên tiền tố vừa dùng. Việc gửi lại cả khối sẽ tạo bộ đơn mới. Với 20 phản hồi có thời gian, P95 theo phương pháp nearest rank là mẫu thứ 19 sau khi sắp tăng dần. Nếu có lỗi mạng, số mẫu có thời gian có thể ít hơn 20.

```powershell
Get-Date -Format o
node .\tools\report.mjs $table "$runId-async-measured" 20
```

## Đọc hai chênh lệch trong bảng gốc

Độ trễ API trung bình giảm từ 229,55 xuống 109,28 ms, chênh 120,27 ms trong lần đo. Ở chiều xử lý phía sau, khoảng hoàn tất xấp xỉ trung bình tăng từ 122,75 lên 445,55 ms, chênh 322,80 ms. Hai kết quả có thể đồng thời đúng vì API bất đồng bộ phản hồi trước khi workflow hoàn tất. Cả hai nhóm đều tìm thấy 20 đơn DONE nên nhận xét này áp dụng cho bộ 20/20 của từng nhánh. Khi thay mức đồng thời hoặc khi SES đang throttle, cần một bảng mới với điều kiện và thời điểm chốt riêng.
