---
title: "Triển khai lại"
weight: 3
chapter: false
pre: " <b> 5.4.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

```powershell
sam validate --region ap-southeast-1
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed.' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed.' }
sam deploy --region ap-southeast-1 --no-fail-on-empty-changeset
if ($LASTEXITCODE -ne 0) { throw 'Inspect the CloudFormation deployment events.' }
```

![Deploy lần hai](/images/5-Workshop/5.4-Idempotency/deploy-result.png)

![Trigger SQS của Lambda đang bật](/images/5-Workshop/5.4-Idempotency/sqs-trigger-enabled.png)

## Xác nhận bản cập nhật đang đọc đúng hàng đợi

Sau khi deploy trên cùng stack, mở **CloudFormation → Resources**. Hai queue đã có ở mục 5.3 phải còn thuộc stack. Lần này có thêm bảng, hàm và event source mapping. Bản build là đầu vào của lần deploy, vì vậy sửa `src/index.mjs` sau khi build đòi hỏi build lại trước khi triển khai thay đổi đó.

Khối sau tìm tên vật lý của hàm từ logical ID `ProcessOrderFn`. Lệnh thứ hai đọc các mapping của chính hàm đó. Dấu hiệu sẵn sàng là một mapping `Enabled` với ARN nguồn kết thúc bằng tên của OrdersQueue và batch bằng 1.

```powershell
$worker = aws cloudformation describe-stack-resource --stack-name order-system --logical-resource-id ProcessOrderFn --region ap-southeast-1 --query 'StackResourceDetail.PhysicalResourceId' --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'ProcessOrderFn was not found.' }
aws lambda list-event-source-mappings --function-name $worker --region ap-southeast-1 --query 'EventSourceMappings[].{State:State,Source:EventSourceArn,Batch:BatchSize}' --output table --no-cli-pager
```

## Nếu trigger chưa sẵn sàng

Ngay sau deploy, mapping có thể còn đang chuyển từ `Creating` hoặc `Enabling`. Chờ ngắn rồi đọc lại cùng mapping. Nếu không có kết quả, xem stack Events và so sánh tên profile, Region và tên hàm. Nếu state đã `Enabled` nhưng chưa có invocation, hãy kiểm tra hàng chính có dữ liệu hay chưa. Bước kiểm thử kế tiếp sẽ gửi một mã đơn riêng để xác nhận toàn bộ đường đi.

Stack đạt `UPDATE_COMPLETE` là một mốc triển khai. Một log mang mã đơn mới và một item có trạng thái `PROCESSING` mới cung cấp thêm hai mốc ở mức ứng dụng. Ghi riêng ba mốc giúp chẩn đoán được tình huống hạ tầng đúng nhưng handler đọc sai dữ liệu.

### Đối chiếu tên bảng của worker

Trong **Lambda → Configuration → Environment variables**, lấy giá trị `TABLE_NAME`. Trong CloudFormation mở `Outputs → TableName`. Khi hai giá trị khớp, phép đọc bảng của bước tiếp theo sẽ truy vấn đúng nơi worker đang ghi. Khi chúng khác nhau, kiểm tra bạn có đang xem hàm từ một stack cũ ở cùng Region hay không. Tên logic `OrdersTable` trong template chỉ là khóa tham chiếu khi triển khai, còn ứng dụng sử dụng tên vật lý.
