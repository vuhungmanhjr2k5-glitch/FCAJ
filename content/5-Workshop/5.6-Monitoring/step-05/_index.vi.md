---
title: "Kết quả giám sát và cảnh báo"
weight: 5
chapter: false
pre: " <b> 5.6.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Mã có trong thông điệp thử: **mon56-20261114-040056-316515**. Dashboard/alarm dùng UTC; hộp thư và thời gian SQS trong ảnh dùng UTC+07:00. Ví dụ 13/11/2026 21:03:44 UTC tương ứng 14/11/2026 04:03:44 UTC+07:00.

**Dashboard bổ sung.** Có đủ bốn widget và legend; DLQ đã có giá trị 1 trong khoảng trước thử. Tại thời điểm chụp dashboard, biểu đồ Step Functions hiển thị ExecutionsFailed = 0; trang chi tiết alarm ghi nhận giá trị 1 ở lần chụp khác. Dashboard này chưa phản ánh kết quả cuối của mọi execution.

![Dashboard bổ sung](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h5.png)

**Step Functions.** Alarm order-monitoring-evidence-sfn-failed ở ALARM, biểu đồ ExecutionsFailed lên 1.

[Alarm Step Functions](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h6.png)

**Lambda.** Alarm order-monitoring-evidence-lambda-errors ở ALARM, biểu đồ Errors lên 1. Riêng ảnh metric không chỉ ra nguyên nhân hoặc MessageId của invocation lỗi.

[Alarm Lambda Errors](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h7.png)

**DLQ.** Đúng queue order-orchestration-evidence-OrdersDLQ-Kbv9OO2FitbU chứa body not-json-mon56-20261114-040056-316515. Ảnh đồng thời có thông điệp cũ; Receive count không được dùng làm số lần Lambda xử lý chính xác vì thao tác nhận ở console cũng ảnh hưởng bộ đếm.

[Thông điệp thử trong DLQ](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h8.png)

**Email ALARM Step Functions.** Email ghi datapoint 1 tại 13/11/2026 21:02 UTC và chuyển ALARM lúc 21:03:44 UTC. Đây là thông báo CloudWatch thực nhận. Chưa có mốc execution thất bại trong bộ ảnh này để tính độ trễ phát hiện.

[Email ALARM Step Functions](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h9.png)

**Email ALARM của Lambda.** Email từ CloudWatch ghi nhận datapoint Errors = 1 tại 13/11/2026 21:01:00 UTC, vượt ngưỡng 0. Alarm order-monitoring-evidence-lambda-errors chuyển từ OK sang ALARM lúc 21:02:21 UTC, tương ứng 14/11/2026 04:02:21 UTC+07:00. Email trong hộp thư xác nhận thông báo ALARM đã được nhận, phù hợp với trạng thái alarm trên CloudWatch. Mốc datapoint không phải thời điểm chính xác Lambda phát sinh lỗi, nên chưa dùng để tính độ trễ phát hiện lỗi hoặc độ trễ gửi email.

[Email ALARM Lambda](/images/5-Workshop/5.6-Monitoring/lambda-alarm-20261114-044826-a961f7/h10.png)

**Email ALARM DLQ.** Email ghi datapoint 1 tại 13/11/2026 20:59 UTC, chuyển ALARM lúc 21:01:06 UTC. Datapoint này có trước lần gửi thông điệp mới trong DLQ của lần thử này; diễn giải là cảnh báo backlog đã tồn tại, không quy chuyển trạng thái này cho riêng thông điệp mới.

[Email cảnh báo DLQ tồn đọng](/images/5-Workshop/5.6-Monitoring/evidence-20261114-043035-985e31/h11.png)

## Dựng lại thứ tự từ các mốc đã ghi

Sắp theo UTC trước khi suy luận. ALARM DLQ lúc 21:01:06 UTC thuộc một datapoint đã xuất hiện trước tin mới. ALARM Lambda lúc 21:02:21 UTC cho biết bộ đo thấy Errors vượt ngưỡng. ALARM Step Functions lúc 21:03:44 UTC cho biết một kỳ có ExecutionsFailed bằng 1. Thứ tự này mô tả hoạt động của ba alarm, còn quan hệ giữa một invocation và một thông điệp cần thêm dấu nhận diện từ ứng dụng.

Ví dụ, body có chuỗi `mon56-20261114-040056-316515` kết nối trực tiếp lần thử với hàng lỗi trong bộ dữ liệu. Ngược lại, đồ thị Errors chỉ biểu diễn số lượng tổng hợp. Một đồ thị tăng cùng vài phút có thể là đầu mối để mở log, nhưng chưa đủ để gán một MessageId cho điểm đó. Khi dashboard và trang Details ở hai thời điểm khác nhau, giữ cả hai mốc để tránh kết luận mâu thuẫn.

## Bảng ghi cho lượt chạy riêng

| Điều cần ghi | Giá trị lấy ở đâu |
|---|---|
| Run ID và dấu hiệu body hỏng | `run.json` trong thư mục lần chạy |
| Tình trạng DLQ trước thử | `dlq-before.json` |
| Tên alarm và trạng thái trước/sau | Baseline, alarm history và email thực nhận |
| Lỗi đầu của workflow | Execution theo đúng mã `-ses-fail` |
| Mốc còn thiếu | Ghi rõ nguồn chưa có, chẳng hạn thời gian email đến |

Timestamp của datapoint đánh dấu kỳ thống kê. Muốn tính độ trễ phát hiện, cần thời gian lỗi thực tế và thời gian alarm đổi trạng thái trên cùng trục thời gian. Muốn tính độ trễ chuyển thư, dùng thời gian nhận thư tương ứng. Bộ kết quả gốc chỉ cho phép kết luận tại các mốc đã quan sát.

### Đọc một email alarm từ trên xuống

Bắt đầu bằng alarm name và Region để chắc chắn đang đọc thông báo của stack đang phân tích. Tiếp theo đọc trạng thái trước, trạng thái sau và lý do chuyển. Phần lý do cho biết datapoint nào đã vượt ngưỡng và kỳ nào được xét. Cuối cùng ghi thời gian chuyển trạng thái. Hộp thư có thể hiển thị giờ theo thiết lập máy, nên nếu muốn so với UTC trong CloudWatch hãy ghi luôn độ lệch múi giờ.

Trong bộ số liệu này, có thể tính khoảng giữa **hai lần chuyển trạng thái** Lambda và Step Functions: từ 21:02:21 đến 21:03:44 là 83 giây. Phép trừ mô tả khoảng cách giữa hai alarm của hai dịch vụ. Nó không xác định được Lambda mất bao lâu để phát hiện lỗi hay một email đi đến hộp thư mất bao lâu. Ví dụ này cho thấy hai timestamp chính xác vẫn có thể trả lời sai câu hỏi nếu chọn nhầm hai sự kiện.

Khi người đọc cần xem thêm, chỉ cần dẫn lại trang chi tiết của đúng alarm hoặc đường dẫn hình đã có cạnh đoạn nhận xét, kèm tên alarm và thời điểm. Nhờ vậy phần kết quả vẫn đọc được như một dòng thời gian cả khi không mở từng hình.
