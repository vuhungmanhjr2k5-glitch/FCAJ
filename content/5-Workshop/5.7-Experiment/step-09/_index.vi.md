---
title: "Kết luận"
weight: 9
chapter: false
pre: " <b> 5.7.9 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Hai phương án đều đạt DONE 20/20 trong lần đo. Các phép thử riêng ghi nhận xử lý gửi trùng, thử lại khi SES lỗi, nhánh cảnh báo SNS và cảnh báo DLQ. Kết luận giới hạn trong các điều kiện và lần thử được minh chứng.

## Quyết định kiến trúc từ hai đường thời gian

Với một khách đang chờ màn hình thanh toán, đại lượng gần trải nghiệm nhất là khoảng từ lúc gửi request đến khi nhận xác nhận tiếp nhận. Trong lượt đã công bố, hai nhóm đều có 20 phản hồi thành công. Trung bình phía client của nhánh đồng bộ là 229,55 ms và của nhánh bất đồng bộ là 109,28 ms. Thông tin giao diện có thể hiển thị ngay ở nhánh bất đồng bộ là mã đơn và trạng thái đã tiếp nhận. Nếu thông báo ngay rằng đơn đã hoàn tất, giao diện sẽ đi trước kết quả của workflow.

Với một nhân viên hỗ trợ tra xem hệ thống đã xử lý đến đâu, cần đọc `DONE` hoặc `FAILED` của đúng mã. Trong nhóm đo, khoảng hoàn tất xấp xỉ trung bình là 122,75 ms cho đồng bộ và 445,55 ms cho bất đồng bộ. Các mốc này đến từ dữ liệu ứng dụng. Việc chúng có thể nhỏ hơn hoặc lớn hơn độ trễ API phía client phụ thuộc điểm bắt đầu, điểm kết thúc và phần xử lý nằm ngoài request. Lấy một chỉ số thay cho chỉ số còn lại sẽ đưa tới quyết định giao diện sai.

## Ghép phép thử với kết luận tương ứng

| Phép thử | Quan sát đã có | Phạm vi của kết luận |
|---|---|---|
| Tải nhỏ ở hai API | 20/20 DONE mỗi phương án, concurrency 1 | Phản hồi API nhanh hơn ở nhánh async trong lần đo đã nêu |
| Năm lần gửi một body | Một execution được xác minh, một tác vụ SES thành công, bộ đếm bằng 1 | Đã kiểm soát trùng cho tình huống và mã đơn đó |
| Email bị từ chối | History ghi lỗi, thử lại và đi qua nhánh báo quản trị | Có thể truy vết thất bại sau khi API đã tiếp nhận |
| Body hỏng | Body xuất hiện trong DLQ, có quan sát alarm | Có đường giữ tin lỗi để điều tra ở tầng nhận |
| Độ trễ báo sự cố | Chưa có đủ cặp timestamp của cùng sự cố | Để mở phép đo này |

Một dòng bảng kết quả nên có cả số yêu cầu đã gửi, số mã đơn riêng biệt, thời điểm chốt và số mẫu thực sự tham gia phép tính. Khi đổi recipient từ simulator sang hộp thư thật hoặc đổi số worker đồng thời, hãy ghi thành lượt độc lập. Giữ bộ số liệu ban đầu làm mốc tham khảo thay vì ghi đè nó.

## Bàn giao kết quả và kết thúc lượt thực hành

Chọn một mã đơn thành công và một mã đơn thất bại từ các lượt bạn đã chạy. Với mỗi mã, ghi đường API hoặc nguồn SQS, trạng thái cuối, bằng chứng đã đọc và điều còn chưa xác nhận. Với thông điệp hỏng không có mã đơn, dùng body marker và tên DLQ. Tập hợp những dòng này cùng bảng độ trễ để người đọc khác có thể lần lại kết luận mà không cần truy cập vào tài khoản của bạn.

Trước khi dọn tài nguyên theo 5.8, lưu lại các prefix, kết quả công cụ đo và thời điểm quan sát cuối. Điều này đặc biệt quan trọng với log group và bảng thuộc stack sắp xóa. Chi phí cần kiểm tra theo thời gian phát sinh mới. Mức budget đã cấu hình hoặc việc một lần đọc hóa đơn hiển thị giá trị nhỏ không thay được phép đo đó.

### Nếu mục tiêu kinh doanh thay đổi

Giả sử người bán chỉ cần báo “đã nhận yêu cầu” trong lúc khách còn ở trang thanh toán. Khi đó có thể dùng tiêu chí API phản hồi và mã đơn truy vết được cho màn hình đầu, rồi để khách tra cứu trạng thái sau. Nếu người bán lại phải xác nhận “đã gửi thư và ghi hoàn tất” trước khi hiển thị màn hình thành công, thời gian cần đánh giá phải kéo dài tới mốc hoàn tất. Bảng đo hiện có cung cấp số liệu ở cả hai mốc để thảo luận lựa chọn này.

Một yêu cầu khác có thể là người trực nhận được cảnh báo trong một khoảng thời gian quy định. Các email đã nhận chứng minh kênh có hoạt động ở lượt quan sát, nhưng dữ liệu thời gian gốc còn thiếu cho phép đo từ lỗi thật. Khi viết kết luận, nêu rõ yêu cầu nào đã được thử và yêu cầu nào cần thiết kế thêm phép đo. Có thể dùng những tài nguyên hiện có cho lần quan sát kế tiếp bằng cách ghi timestamp ngay khi phát sinh lỗi, rồi ghép với change history và thư nhận cùng tên alarm.

### Một đoạn kết quả có thể điền từ lần chạy mới

“Lượt … tại Singapore dùng … đơn riêng biệt cho mỗi phương án và concurrency …. Trung bình/P95 của API là … với đồng bộ và … với bất đồng bộ. Tại thời điểm … UTC, bảng có … đơn DONE, … đơn FAILED và … đơn đang chờ. Đơn … minh họa đường lỗi ở bước …. Chỉ số … còn thiếu nguồn hoặc mốc ….” Điền từ kết quả đo, giữ nguyên ô chưa xác nhận. Đoạn này đặt số liệu, điều kiện và giới hạn trong một chỗ để người đọc có thể so trực tiếp với lượt gốc.
