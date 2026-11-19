---
title: "Đặt cảnh báo chi phí"
weight: 5
chapter: false
pre: " <b> 5.2.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

1. Vào *Billing and Cost Management* → *Budgets* → *Create budget*.
2. Chọn *Zero spend budget* hoặc *Monthly cost budget* (ví dụ 5 USD).
3. Nhập email nhận cảnh báo.

![Tạo budget](/images/5-Workshop/5.2-Prerequisite/004.png)

<!-- workshop-detail:v1:begin -->
## Mục đích của Budget

AWS Budgets giúp theo dõi chi phí so với ngưỡng do người thực hành đặt. Ngưỡng cần phù hợp ngân sách cá nhân; một giá trị trong ví dụ chỉ là tham khảo, không phải ước tính chắc chắn về tổng chi phí workshop.

Thông báo Budget có độ trễ và không tự dừng toàn bộ tài nguyên khi vượt ngưỡng. Vì vậy, cần kết hợp cảnh báo với việc kiểm tra tài nguyên và dọn dẹp sau thực hành.

## Những thông tin cần kiểm tra

1. Loại budget và kỳ theo dõi, chẳng hạn ngân sách chi phí theo tháng.
2. Mức ngân sách phù hợp và ngưỡng cảnh báo đã chọn.
3. Địa chỉ email nhận thông báo được nhập chính xác.
4. Phạm vi lọc: toàn tài khoản hay một nhóm tài nguyên/tag đã được cấu hình.

Nếu đang xem chi phí toàn tài khoản, số liệu có thể bao gồm hoạt động ngoài workshop. Khi chụp Cost Explorer cần giữ khoảng ngày và bộ lọc để người đọc hiểu phạm vi. Dữ liệu chưa cập nhật hoặc số hiển thị đã làm tròn không đủ để kết luận bài lab hoàn toàn miễn phí.

## Sau buổi thực hành

Ghi lại các stack, bucket, log group và tài nguyên được giữ lại. Chỉ dọn dẹp khi đã lưu minh chứng và không còn cần website để demo. Không tạo thêm lượt gửi email hoặc tài nguyên chỉ để làm số chi phí xuất hiện trên biểu đồ.

<!-- workshop-detail:v1:end -->

<!-- workshop-proof:v1:begin -->
## Minh chứng thiết lập ngân sách

{{< workshop-proof file="Budget.png" alt="Danh sách AWS Budgets và ngân sách workshop" caption="Ngân sách order-workshop-budget được đặt ở mức 5 USD; danh sách hiển thị Thresholds OK và Health status Healthy." >}}

Giá trị 5 USD là mức ngân sách đã cấu hình, không phải số tiền đã chi. My Monthly Cost Budget ở mức 100 USD là một budget khác trong cùng danh sách. Ảnh chưa thể hiện kỳ theo dõi, ngưỡng thông báo chi tiết, email nhận hoặc việc cảnh báo đã được gửi. Mở chi tiết order-workshop-budget nếu cần kiểm tra các thông tin này.
<!-- workshop-proof:v1:end -->

## Đọc ngưỡng trong một ngân sách tháng

Giả sử bạn chọn ngân sách tháng 5 USD và một cảnh báo khi chi phí thực tế đạt 80%. Ngưỡng tương ứng là 4 USD trong phạm vi đã chọn. Nếu chọn **Forecasted**, AWS đánh giá chi phí dự báo cho cả kỳ. Cùng một tỷ lệ 80% vì vậy có thể dẫn đến thông báo ở thời điểm khác nhau. Hãy ghi rõ lựa chọn **Actual** hay **Forecasted** bên cạnh số tiền.

Nếu tài khoản có dự án khác, trước tiên nhìn bộ lọc của budget. Khi không lọc, bức thư vượt ngưỡng của cả tài khoản chưa cho biết stack nào gây tăng phí. Mở Cost Explorer, chọn đúng khoảng ngày, nhóm theo dịch vụ rồi xem dịch vụ liên quan đến hoạt động nào của buổi học. Ví dụ, số lần gọi Lambda có thể tăng khi thông điệp hỏng quay lại hàng đợi.

## Sau khi nhận một thông báo

Ghi thời điểm nhận email và khoảng thời gian mà budget đang tính. Xem còn phép thử nào đang chạy, còn stack nào dành cho demo, còn log hoặc dashboard nào cần giữ. Nếu vừa xóa tài nguyên, chờ dữ liệu chi phí cập nhật rồi xem phần phát sinh mới. Bản ghi gồm kỳ, ngưỡng, phạm vi và quyết định tiếp theo đủ để người khác hiểu bạn đã phản ứng thế nào.
