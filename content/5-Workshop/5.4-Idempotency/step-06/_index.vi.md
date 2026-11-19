---
title: "Xử lý sự cố"
weight: 6
chapter: false
pre: " <b> 5.4.6 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

- **Lambda không chạy**: vào Console → Lambda → hàm `ProcessOrderFn` → tab **Configuration → Triggers**, kiểm tra trigger SQS đang ở trạng thái *Enabled*.
- **Lỗi quyền DynamoDB**: xem log, kiểm tra phần `Policies` trong template đã dùng đúng `!Ref OrdersTable`.
- **`sam logs` không hiện gì**: chờ vài giây rồi gửi lại tin nhắn, log có độ trễ nhỏ.

## Thu hẹp phạm vi từ queue đến bảng

Bắt đầu bằng một đơn có mã mới và ghi thời gian gửi ở UTC. Xác nhận API SQS đã trả MessageId. Sau đó xem invocation của đúng hàm trong cùng khoảng thời gian. Nếu queue còn tin visible mà mapping không hoạt động, kiểm tra event source mapping. Nếu queue trở nên ít tin visible nhưng số tin in-flight tăng, worker có thể đang xử lý hoặc chờ hết visibility timeout sau một lỗi.

```powershell
sam logs --stack-name order-system --name ProcessOrderFn --region ap-southeast-1 --start-time '15min ago'
```

Trong output, tìm lần xuất hiện đầu của mã đơn hoặc exception gần thời điểm gửi. `Runtime.ImportModuleError` gợi ý kiểm tra tệp handler, đường dẫn `CodeUri` và thư viện nằm trong bản build. `AccessDeniedException` khi PutItem dẫn đến role thực thi và ARN bảng. `ConditionalCheckFailedException` đã được xử lý là hành vi dự kiến của yêu cầu trùng. Một log stream có thể chứa nhiều đơn nên luôn giữ mốc thời gian cạnh kết quả.

## Kiểm tra sau khi sửa

Đổi từng yếu tố rồi chạy lại một trường hợp nhỏ. Sửa mã hay template thì build và cập nhật cùng stack. Sửa bộ lọc log thì chỉ cần đọc lại log. Khi muốn kiểm tra nhánh “đơn mới”, dùng mã mới để không vấp điều kiện của item cũ. Khi muốn kiểm tra nhánh “trùng”, cố ý giữ cùng mã. Ghi triệu chứng ban đầu, thay đổi đã làm và một kết quả sau sửa. Ba dòng đó mô tả được nguyên nhân và cách kiểm tra lại mà không cần thêm hình.
