---
title: "Đánh giá và giới hạn"
weight: 6
chapter: false
pre: " <b> 5.6.6 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

| Nội dung | Kết quả được minh chứng |
|---|---|
| Dashboard | Có bốn nhóm biểu đồ |
| Alarm Step Functions / Lambda | Cả hai được ghi nhận ALARM |
| Thông điệp thử vào DLQ | Có body và queue trùng lần thử |
| Email ALARM Step Functions / DLQ | Đã nhận; DLQ gắn với backlog |
| Email ALARM Lambda | Đã nhận thông báo CloudWatch ALARM, khớp tên alarm và datapoint Errors = 1 |
| Phục hồi và độ trễ cảnh báo | Chưa đủ bằng chứng để kết luận hoặc tính thời gian |

Polling DLQ làm thông điệp tạm invisible, có thể giảm visible count dù chưa xử lý/xóa. OK với notBreaching không tự chứng minh ứng dụng khỏe. Giữ run.json, sent.json, alarms-before.json và dlq-before.json để đối chiếu; không suy ra tổng số đơn hay kết quả từng đơn chỉ từ metric.

## Xem lại tình huống alarm về OK

Giả sử người vận hành mở DLQ và poll tin để xem body. Trong thời gian ẩn, số tin visible có thể giảm trong khi tin vẫn còn trong queue. Nếu không có datapoint vi phạm ở kỳ kế tiếp, alarm có thể về `OK`. Để xác nhận phục hồi nghiệp vụ, cần kiểm tra tin đã được xử lý thế nào, item cuối mang trạng thái gì và có tái xuất hiện sau visibility timeout hay không. Một mình chuyển trạng thái alarm chưa trả lời đủ các câu đó.

Tương tự, một khoảng không có Lambda Errors có thể do không có dữ liệu đầu vào. Kiểm tra thêm Invocations và khả năng truy vết một đơn hợp lệ trong chính khoảng đó. Với Step Functions, một execution đang Retry chưa được tính là execution Failed cho đến khi đi tới trạng thái kết thúc phù hợp.

## Khoanh phạm vi nhận xét trong báo cáo

Tách bài nhận xét thành ba câu. Câu đầu nêu điều trực tiếp quan sát, ví dụ đúng body thử xuất hiện trong đúng DLQ. Câu sau nêu giả thuyết cần thêm dữ liệu, ví dụ một datapoint Errors có thể liên quan đến lỗi parse. Câu cuối ghi phép kiểm tra tiếp theo có thể phân biệt, ví dụ tìm log theo timestamp và đối chiếu MessageId. Cấu trúc này có thể dùng ngay cả khi chỉ có bản ghi văn bản.

Nếu chưa ghi được mốc lỗi gốc, hãy để trống độ trễ cảnh báo. Nếu chưa quan sát recovery, giữ trạng thái “chưa kiểm tra”. Các giới hạn này giúp người đọc biết cần thu thêm dữ liệu nào trước khi đưa ra cam kết vận hành.

### Lập kế hoạch kiểm tra phục hồi cho một lần sau

Trước một phép thử phục hồi, ghi rõ cơ chế nào cần hồi phục. Với lỗi SES, mục tiêu có thể là một **đơn mới** dùng email hợp lệ đi đến `DONE` sau khi sửa identity. Với DLQ, mục tiêu có thể là một thông điệp riêng đã được xem, sửa dữ liệu ở nguồn và xử lý theo quy trình của ứng dụng. Một chuỗi văn bản không phải JSON vẫn sẽ lỗi nếu chỉ đưa nguyên văn về hàng chính.

Ở mỗi trường hợp, chọn mốc đầu, hành động tác động và dấu hiệu cuối có thể kiểm tra. Ví dụ, ghi mã đơn lỗi, thay địa chỉ nhận trong đầu vào thử mới, tạo mã đơn khác rồi đọc kết quả gửi thư và cập nhật bảng. Lưu rõ đây là kiểm chứng cho yêu cầu mới sau sửa lỗi. Bản ghi của đơn cũ vẫn mô tả lần thất bại ban đầu.

| Câu hỏi còn mở | Dữ liệu tối thiểu cho lần đo sau |
|---|---|
| Từ lỗi tới phát hiện mất bao lâu? | Lỗi gốc có timestamp và lần chuyển alarm tương ứng |
| Hộp thư nhận chậm bao lâu? | Cùng thông báo, mốc chuyển và mốc nhận ở inbox |
| Backlog đã xử lý xong chưa? | Dấu nhận diện từng tin, trạng thái xử lý cuối và quan sát sau visibility timeout |

Bảng này là kế hoạch thu dữ liệu. Các kết quả phục hồi sẽ chỉ được điền khi thực sự thực hiện lượt đo đó.
