---
title: "Kiểm tra kết quả"
weight: 5
chapter: false
pre: " <b> 5.5.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

| Trường hợp | Kết quả mong đợi |
|---|---|
| Đơn hợp lệ | Email đến khách, execution Succeeded, trạng thái `DONE` |
| Gửi trùng | Không có email thứ hai, log báo trùng |
| Email không gửi được | Thử lại 3 lần, execution Failed, trạng thái `FAILED`, admin nhận cảnh báo |

## Từ một mã đơn đến một kết luận

Lấy tên execution từ lần gửi ở trang 5.5.4, đọc trạng thái cuối và tìm đúng item theo `orderId`. Nếu chỉ xem toàn bộ bảng bằng Scan, bản ghi mới dễ bị lẫn với đơn của lượt khác. Mở cùng mã trong hai dịch vụ giúp biết workflow đã kết thúc chưa và bảng đang phản ánh kết quả nào. Với người nhận thật, nối thêm tiêu đề thư có mã đơn và thời điểm nhận.

Đối với thử trùng, ghi số lần gửi lại body và thời điểm tương đối so với lúc đơn đầu đã `DONE`. Gửi lại trong khi execution còn chạy sẽ chạm nhánh khác gửi lại sau khi đã kết thúc. Cả hai có thể dẫn đến một execution hợp lệ nhưng log khác nhau. Khi viết kết quả, nêu rõ thời điểm bạn thực sự đã thử.

## Đọc trạng thái còn dang dở

| Quan sát | Ý nghĩa có thể kiểm tra | Bước đọc tiếp |
|---|---|---|
| Item `RECEIVED`, chưa thấy execution | Có bản ghi đầu nhưng chưa xác nhận đã khởi động | Log hàm và quyền StartExecution |
| Execution đang chạy | Có thể đang chờ retry | Mở sự kiện cuối và thời gian thử kế |
| Item `FAILED`, email chưa tới | Ghi bảng đã hoàn thành, kênh báo còn cần kiểm tra | Kết quả NotifyAdminFailure và subscription |
| Item `DONE`, dùng simulator | Workflow đã đi đường thành công | Đọc sự kiện SES, ghi rõ loại người nhận |

Một kết quả hoàn chỉnh cho phần này có thể là bảng ba dòng gồm mã đơn, kết quả workflow, trạng thái bảng và quan sát thư. Nếu một nguồn chưa có dữ liệu, ghi thời điểm kiểm tra thay cho suy đoán.

### Tra đúng execution theo tên đơn

Đối với Standard workflow trong template, tên execution là mã đơn. Nhập mã của lượt thử rồi đọc trạng thái, thời điểm bắt đầu và kết thúc. Nếu AWS trả ExecutionDoesNotExist, kiểm tra tên và stack trước khi kết luận worker chưa khởi động.

```powershell
$orderId = Read-Host 'Order ID'
$sm = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --query "Stacks[0].Outputs[?OutputKey=='StateMachineArn'].OutputValue | [0]" --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $sm -eq 'None') { throw 'StateMachineArn unavailable.' }
$executionArn = ($sm -replace ':stateMachine:', ':execution:') + ':' + $orderId
aws stepfunctions describe-execution --execution-arn $executionArn --region ap-southeast-1 --query '{State:status,Start:startDate,End:stopDate}' --output table --no-cli-pager
```
