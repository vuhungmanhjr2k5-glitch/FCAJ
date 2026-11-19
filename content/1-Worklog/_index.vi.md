---
title: "Nhật ký công việc"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b> 1. </b> "
---

Trong thời gian thực tập *12 tuần*, từ *10/08/2026 đến 31/10/2026*, tôi tập trung tìm hiểu các dịch vụ AWS và vận dụng vào đề tài *Xây dựng luồng xử lý đơn hàng theo kiến trúc event-driven, có khả năng chịu lỗi và quan sát; đánh giá so sánh với phương án xử lý đồng bộ*.

Trong *8 tuần đầu*, tôi xây dựng kiến thức nền tảng về bảo mật, phát triển API, xử lý bằng AWS Lambda, lưu trữ trạng thái đơn hàng và trao đổi thông điệp giữa các thành phần. Song song với việc thực hành, tôi tìm hiểu cách ghi nhận log, theo dõi lỗi và thiết kế các tình huống kiểm thử phục vụ việc đánh giá hệ thống.

Trong *4 tuần cuối*, tôi tập trung tích hợp luồng xử lý đơn hàng, kiểm tra cơ chế retry và xử lý thông điệp trùng, đánh giá khả năng phục hồi khi thành phần xử lý gặp lỗi, đồng thời thực hiện các kịch bản so sánh với phương án đồng bộ.

Việc đánh giá phân biệt thời gian API tiếp nhận yêu cầu với thời gian hoàn tất đơn hàng. Các tiêu chí được xem xét gồm độ trễ, thông lượng hoàn tất, tỷ lệ thành công, tình trạng tồn đọng và khả năng phục hồi sau sự cố. Kết luận được giới hạn trong cấu hình và điều kiện thử nghiệm thực tế.

Phần công việc cá nhân tôi phụ trách: *[Bổ sung các thành phần đã triển khai, kiểm thử hoặc viết tài liệu trực tiếp.]*

Lịch làm việc theo từng tuần như sau:

*Tuần 1 (10/08 – 15/08/2026):* [Thiết lập môi trường AWS và xác định bài toán xử lý đơn hàng.](1.1-week1-worklog/)

*Tuần 2 (17/08 – 22/08/2026):* [Thiết kế API và tìm hiểu luồng xử lý đồng bộ.](1.2-week-2-worklog/)

*Tuần 3 (24/08 – 29/08/2026):* [Thực hành AWS Lambda và lưu trữ trạng thái đơn hàng.](1.3-week-3-worklog/)

*Tuần 4 (31/08 – 05/09/2026):* [Xây dựng phương án xử lý đồng bộ làm cơ sở đối chiếu.](1.4-week-4-worklog/)

*Tuần 5 (07/09 – 12/09/2026):* [Thiết kế sự kiện và thực hành định tuyến với Amazon EventBridge.](1.5-week-5-worklog/)

*Tuần 6 (14/09 – 19/09/2026):* [Thực hành Amazon SQS, retry và hàng đợi lỗi.](1.6-week-6-worklog/)

*Tuần 7 (21/09 – 26/09/2026):* [Xây dựng khả năng quan sát luồng xử lý đơn hàng.](1.7-week-7-worklog/)

*Tuần 8 (28/09 – 03/10/2026):* [Thiết kế dữ liệu thử nghiệm và phương pháp đo đạc.](1.8-week-8-worklog/)

*Tuần 9 (05/10 – 10/10/2026):* [Tích hợp luồng xử lý đơn hàng theo sự kiện.](1.9-week-9-worklog/)

*Tuần 10 (12/10 – 17/10/2026):* [Kiểm thử khả năng chịu lỗi và xử lý thông điệp trùng.](1.10-week-10-worklog/)

*Tuần 11 (19/10 – 24/10/2026):* [Đo đạc và so sánh hai phương án xử lý.](1.11-week-11-worklog/)

*Tuần 12 (26/10 – 31/10/2026):* [Hoàn thiện báo cáo, minh chứng và tài liệu hướng dẫn.](1.12-week-12-worklog/)