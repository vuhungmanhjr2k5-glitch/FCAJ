---
title: "Bản đề xuất"
date: 2026-10-05
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## 1. Tên đề tài

**Xây dựng luồng xử lý đơn hàng theo kiến trúc event-driven, có khả năng chịu lỗi và quan sát trên AWS; đánh giá so sánh với phương án xử lý đồng bộ.**

## 2. Bối cảnh và vấn đề cần giải quyết

Trong một ứng dụng thương mại điện tử, việc tiếp nhận đơn hàng có thể kéo theo nhiều bước xử lý như kiểm tra dữ liệu, ghi nhận đơn hàng, cập nhật trạng thái và gửi thông báo.

Với phương án đồng bộ, người dùng phải chờ các bước nằm trong luồng yêu cầu hoàn tất trước khi nhận kết quả. Khi một bước xử lý chậm hoặc dịch vụ phụ thuộc gặp sự cố, thời gian phản hồi có thể tăng và yêu cầu có thể vượt quá thời gian chờ.

Việc người dùng gửi lại yêu cầu sau khi gặp timeout cũng tạo ra nguy cơ xử lý trùng nếu hệ thống chưa có cơ chế kiểm soát phù hợp. Tuy nhiên, timeout không đồng nghĩa với việc đơn hàng chắc chắn chưa được xử lý, vì một số bước có thể đã hoàn thành trước khi kết nối bị gián đoạn.

Kiến trúc event-driven cho phép tách bước tiếp nhận đơn hàng khỏi các bước xử lý phía sau thông qua sự kiện hoặc thông điệp. Cách tiếp cận này có thể giúp hệ thống tiếp nhận yêu cầu khi thành phần xử lý đang chậm và thử lại các công việc gặp lỗi tạm thời.

Đổi lại, hệ thống cần xử lý thêm các vấn đề như thông điệp trùng, dữ liệu cập nhật không tức thời, tồn đọng hàng đợi và khó theo dõi luồng xử lý qua nhiều thành phần.

Đề tài được đề xuất nhằm xây dựng hai phương án có cùng phạm vi nghiệp vụ, từ đó đánh giá sự khác biệt bằng các thử nghiệm có kiểm soát.

## 3. Mục tiêu đề tài

### 3.1. Mục tiêu tổng quát

Xây dựng mô hình xử lý đơn hàng trên AWS theo kiến trúc event-driven, có cơ chế xử lý lỗi và theo dõi trạng thái, đồng thời so sánh với phương án đồng bộ dựa trên dữ liệu thực nghiệm.

### 3.2. Mục tiêu cụ thể

- Xây dựng API tiếp nhận đơn hàng và API truy vấn trạng thái.
- Triển khai phương án đồng bộ làm cơ sở đối chiếu.
- Triển khai phương án bất đồng bộ sử dụng hàng đợi và thành phần xử lý độc lập.
- Thiết kế cơ chế retry, dead-letter queue và phục hồi thông điệp lỗi.
- Kiểm soát tác động của yêu cầu hoặc thông điệp được gửi lại bằng idempotency.
- Liên kết log giữa các thành phần để theo dõi quá trình xử lý một đơn hàng.
- Xây dựng dashboard và cảnh báo cho lỗi xử lý, tồn đọng và thông điệp thất bại.
- Đo đạc độ trễ, thông lượng hoàn tất, tỷ lệ thành công và hành vi phục hồi của hai phương án.
- Tổng hợp điều kiện phù hợp, hạn chế và sự đánh đổi của từng kiến trúc.

## 4. Phạm vi triển khai

### 4.1. Chức năng trong phạm vi

Hệ thống sử dụng dữ liệu giả lập để thực hiện các chức năng:

- Tiếp nhận yêu cầu tạo đơn hàng.
- Kiểm tra các trường dữ liệu bắt buộc.
- Thực hiện các bước nghiệp vụ mô phỏng với thời gian xử lý có thể điều chỉnh.
- Ghi nhận và cập nhật trạng thái đơn hàng.
- Cung cấp chức năng truy vấn trạng thái.
- Ghi nhận kết quả thông báo mô phỏng.
- Kiểm thử lỗi, xử lý trùng và khả năng phục hồi.

Các trạng thái dự kiến gồm:

| Trạng thái | Ý nghĩa |
|------------|---------|
| ACCEPTED | Đơn hàng đã được tiếp nhận theo cơ chế lưu bền vững được lựa chọn. |
| PROCESSING | Đơn hàng đang được xử lý. |
| COMPLETED | Các bước nghiệp vụ bắt buộc đã hoàn tất. |
| FAILED | Đơn hàng được xác định thất bại theo quy tắc xử lý. |

Thông điệp nằm trong dead-letter queue sẽ được theo dõi riêng để điều tra và phục hồi. Việc chuyển thông điệp vào hàng đợi lỗi cần được đối chiếu với trạng thái nghiệp vụ, không mặc định coi mọi trường hợp là đơn hàng đã thất bại vĩnh viễn.

### 4.2. Giới hạn phạm vi

- Không tích hợp thanh toán thật hoặc sử dụng dữ liệu khách hàng thật.
- Không xây dựng đầy đủ một nền tảng thương mại điện tử.
- Thử nghiệm trong một AWS Region với giới hạn tài nguyên được ghi nhận.
- Không đặt mục tiêu chứng minh khả năng vận hành ở quy mô sản xuất.
- Kết luận chỉ áp dụng cho phạm vi nghiệp vụ, cấu hình và mức tải đã kiểm thử.

## 5. Kiến trúc và giải pháp đề xuất

### 5.1. Phương án xử lý đồng bộ

Luồng xử lý dự kiến:

**Client → API Gateway → Lambda xử lý nghiệp vụ → DynamoDB → Phản hồi kết quả**

Hàm xử lý thực hiện các bước nghiệp vụ trong phạm vi một yêu cầu và trả kết quả sau khi hoàn tất hoặc gặp lỗi.

Phương án này được sử dụng làm cơ sở đối chiếu về thời gian phản hồi, thời gian hoàn tất và hành vi khi có lỗi.

### 5.2. Phương án xử lý theo sự kiện

Luồng xử lý logic dự kiến:

**Client → API Gateway → Lambda tiếp nhận → SQS → Lambda xử lý → DynamoDB**

Các thành phần hỗ trợ:

- **SQS dead-letter queue:** lưu thông điệp vượt quá ngưỡng xử lý lại.
- **CloudWatch:** thu thập log, metric, dashboard và cảnh báo.
- **API truy vấn trạng thái:** cung cấp tiến độ và kết quả xử lý theo mã đơn hàng.

API tiếp nhận trả về mã đơn hàng cùng phản hồi tiếp nhận sau khi yêu cầu đã được đưa vào cơ chế lưu bền vững đã thiết kế. Phản hồi này không có nghĩa là toàn bộ nghiệp vụ đã hoàn tất.

Nếu cần lưu trạng thái tiếp nhận vào DynamoDB đồng thời phát thông điệp, thiết kế phải giải quyết trường hợp một thao tác thành công nhưng thao tác còn lại thất bại. Một phương án được xem xét là transactional outbox: ghi đơn hàng và bản ghi công việc trong cùng giao dịch, sau đó chuyển công việc sang hàng đợi bằng bộ chuyển tiếp có cơ chế thử lại và đối soát.

Nếu thông điệp được đưa trực tiếp vào SQS trước khi consumer tạo bản ghi đơn hàng, API truy vấn phải mô tả rõ khoảng thời gian trạng thái chưa xuất hiện trong cơ sở dữ liệu.

Lựa chọn cuối cùng sẽ được ghi rõ trong tài liệu triển khai và kiểm thử tích hợp.

### 5.3. Vai trò của các dịch vụ AWS

| Dịch vụ | Vai trò dự kiến |
|---------|----------------|
| Amazon API Gateway | Cung cấp API tạo đơn hàng và truy vấn trạng thái. |
| AWS Lambda | Kiểm tra yêu cầu, xử lý nghiệp vụ và cập nhật trạng thái. |
| Amazon SQS | Lưu đệm công việc và tách tốc độ tiếp nhận khỏi tốc độ xử lý. |
| Amazon SQS DLQ | Cô lập thông điệp xử lý thất bại nhiều lần. |
| Amazon DynamoDB | Lưu trạng thái đơn hàng và dữ liệu phục vụ idempotency. |
| Amazon CloudWatch | Thu thập log, theo dõi metric và phát cảnh báo. |
| AWS IAM | Quản lý quyền truy cập giữa các thành phần theo nguyên tắc quyền tối thiểu. |

Các dịch vụ định tuyến hoặc thông báo bổ sung chỉ được đưa vào khi có yêu cầu cụ thể, tránh mở rộng kiến trúc vượt quá phạm vi đánh giá.

## 6. Thiết kế khả năng chịu lỗi và quan sát

### 6.1. Retry và dead-letter queue

Hệ thống phân biệt lỗi tạm thời với lỗi dữ liệu hoặc lỗi nghiệp vụ không thể khắc phục bằng cách thử lại.

Các nội dung cần cấu hình và kiểm tra gồm:

- Timeout của hàm xử lý.
- Visibility timeout của hàng đợi.
- Kích thước batch và giới hạn xử lý đồng thời.
- Ngưỡng số lần nhận thông điệp trước khi chuyển vào DLQ.
- Báo cáo lỗi từng thông điệp trong batch.
- Quy trình kiểm tra, khắc phục nguyên nhân và redrive thông điệp lỗi.

Thông điệp không được đưa trở lại luồng xử lý một cách tự động, không giới hạn khi nguyên nhân lỗi chưa được xác định.

### 6.2. Idempotency và tính nhất quán

SQS Standard có thể giao thông điệp nhiều lần và không bảo đảm thứ tự tuyệt đối. Vì vậy, consumer cần được thiết kế để xử lý việc giao lại mà không tạo thêm tác động nghiệp vụ ngoài ý muốn.

Giải pháp dự kiến:

- Sử dụng khóa idempotency ổn định theo đơn hàng và thao tác nghiệp vụ.
- Dùng ghi có điều kiện để kiểm soát xử lý đồng thời.
- Phân biệt công việc đang xử lý với công việc đã hoàn tất.
- Xác định cách tiếp tục khi hàm bị gián đoạn giữa các bước.
- Kiểm soát chuyển trạng thái để thông điệp cũ không ghi đè trạng thái mới.
- Kiểm thử cả trường hợp thông điệp trùng và trường hợp lỗi xảy ra sau khi một tác động nghiệp vụ đã được thực hiện.

Việc lưu một khóa chống trùng không tự động bảo đảm mọi tác động bên ngoài chỉ xảy ra đúng một lần. Những giới hạn này phải được nêu rõ trong báo cáo.

### 6.3. Log, metric và cảnh báo

Log dự kiến sử dụng định dạng có cấu trúc với các trường:

- orderId.
- eventId hoặc messageId.
- correlationId.
- Tên bước xử lý.
- Trạng thái và loại lỗi.
- Thời điểm bắt đầu, kết thúc và thời gian xử lý.

Dashboard tập trung vào:

- Số yêu cầu tiếp nhận và số đơn hàng hoàn tất.
- Lambda errors, duration và throttles.
- Số thông điệp tồn đọng và tuổi thông điệp lâu nhất.
- Số thông điệp trong DLQ.
- Thời gian hoàn tất đơn hàng và tỷ lệ lỗi.

Các số liệu hàng đợi của SQS được sử dụng để giám sát xu hướng; việc đối soát số đơn hàng chính xác dựa trên mã định danh và dữ liệu thử nghiệm.

## 7. Phương pháp thử nghiệm và đánh giá

### 7.1. Nguyên tắc so sánh

Hai phương án sử dụng cùng phạm vi nghiệp vụ, cùng bộ dữ liệu đầu vào và cùng định nghĩa về một đơn hàng hoàn tất.

Cấu hình tài nguyên, giới hạn đồng thời, chính sách retry, phiên bản mã nguồn và điều kiện chạy được ghi lại để có thể tái lập thử nghiệm.

Cần phân biệt hai loại thời gian:

- **Thời gian phản hồi API:** từ lúc client gửi yêu cầu đến khi nhận phản hồi.
- **Thời gian hoàn tất đơn hàng:** từ lúc gửi yêu cầu ban đầu đến khi đơn hàng đạt trạng thái kết thúc được xác định.

Phản hồi tiếp nhận nhanh của phương án bất đồng bộ không được sử dụng thay cho thời gian hoàn tất để kết luận hệ thống xử lý nghiệp vụ nhanh hơn.

### 7.2. Các kịch bản thử nghiệm

| Kịch bản | Nội dung đánh giá |
|----------|------------------|
| Tải ổn định | Độ trễ và thông lượng khi tốc độ gửi yêu cầu ít biến động. |
| Tải tăng đột ngột | Khả năng tiếp nhận, mức tồn đọng và thời gian xử lý hết hàng đợi. |
| Một bước xử lý chậm | Ảnh hưởng đến phản hồi API và thời gian hoàn tất. |
| Lỗi tạm thời | Hành vi retry và khả năng hoàn tất sau khi lỗi được gỡ bỏ. |
| Lỗi kéo dài | Cách cô lập thông điệp, cảnh báo và phục hồi bằng DLQ. |
| Gửi lại yêu cầu hoặc thông điệp | Hiệu quả kiểm soát tác động nghiệp vụ trùng. |

### 7.3. Chỉ số đánh giá

- p50 và p95 của thời gian phản hồi API.
- p50 và p95 của thời gian hoàn tất đối với các đơn đã hoàn tất.
- Số đơn hàng duy nhất hoàn tất trong một đơn vị thời gian.
- Tỷ lệ hoàn tất, thất bại và chưa hoàn tất tại thời điểm kết thúc quan sát.
- Mức tồn đọng và thời gian xử lý hết hàng đợi.
- Thời gian phục hồi sau khi nguyên nhân lỗi được loại bỏ.
- Số tác động nghiệp vụ trùng được phát hiện.
- Chi phí thử nghiệm nếu có dữ liệu đủ để phân bổ.

Các đơn thất bại hoặc chưa hoàn tất được báo cáo riêng, tránh loại bỏ chúng khỏi phân tích và tạo ra kết luận sai lệch.

Mỗi kịch bản được chạy lặp lại. Báo cáo sẽ ghi số lần chạy, thời lượng, mức tải và sự biến động giữa các lần đo.

## 8. Kế hoạch thực hiện

Kế hoạch triển khai tập trung trong bốn tuần cuối, sử dụng kiến thức và các thành phần thử nghiệm đã chuẩn bị ở giai đoạn trước.

| Thời gian | Nội dung chính | Sản phẩm dự kiến |
|-----------|----------------|------------------|
| Tuần 9: 05/10–10/10/2026 | Chốt kiến trúc, tích hợp API, hàng đợi, consumer và lưu trữ trạng thái. | Hai luồng xử lý có thể chạy thử; sơ đồ kiến trúc và đặc tả API. |
| Tuần 10: 12/10–17/10/2026 | Hoàn thiện idempotency, retry, DLQ, log và cảnh báo; kiểm thử lỗi. | Biên bản kiểm thử lỗi và hướng dẫn phục hồi. |
| Tuần 11: 19/10–24/10/2026 | Chạy các kịch bản tải, thu thập dữ liệu và so sánh hai phương án. | Dữ liệu thô, bảng tổng hợp và biểu đồ đánh giá. |
| Tuần 12: 26/10–31/10/2026 | Rà soát kết quả, hoàn thiện tài liệu, minh chứng và báo cáo. | Bộ mã nguồn, hướng dẫn triển khai và báo cáo cuối cùng. |

## 9. Kết quả kỳ vọng và tiêu chí nghiệm thu

### 9.1. Sản phẩm bàn giao

- Mã nguồn của phương án đồng bộ và phương án event-driven.
- Sơ đồ kiến trúc và tài liệu mô tả luồng xử lý.
- Hướng dẫn triển khai, cấu hình và dọn dẹp tài nguyên.
- Bộ dữ liệu, công cụ và kịch bản thử nghiệm.
- Dashboard, truy vấn log và cấu hình cảnh báo.
- Biên bản kiểm thử retry, DLQ và idempotency.
- Báo cáo so sánh có dữ liệu và minh chứng.
- Worklog và tài liệu tổng hợp bằng tiếng Việt và tiếng Anh.

### 9.2. Tiêu chí nghiệm thu

- Cả hai phương án thực hiện đúng phạm vi nghiệp vụ đã thống nhất.
- Có thể tra cứu trạng thái và theo dõi quá trình xử lý theo mã đơn hàng.
- Các tình huống lỗi được kiểm tra bằng kịch bản có thể tái lập.
- Quy trình phục hồi thông điệp lỗi được mô tả và thực hành.
- Cơ chế chống xử lý trùng được kiểm chứng trong phạm vi tác động nghiệp vụ đã xác định.
- Kết quả so sánh tách biệt thời gian phản hồi với thời gian hoàn tất.
- Các hạn chế, trường hợp chưa đạt và vấn đề còn tồn tại được ghi rõ.

Đề tài không đặt trước kết luận rằng event-driven luôn tốt hơn xử lý đồng bộ. Kết quả kỳ vọng là xác định được điều kiện phù hợp và sự đánh đổi của từng phương án dựa trên số liệu.

## 10. Chi phí, rủi ro và biện pháp kiểm soát

### 10.1. Kiểm soát chi phí

Chi phí phụ thuộc vào Region, số yêu cầu, thời gian thực thi Lambda, lưu lượng SQS, thao tác DynamoDB và lượng log được lưu.

Trước khi kiểm thử tải, cần:

- Ước tính chi phí theo cấu hình và mức tải dự kiến.
- Thiết lập AWS Budgets và ngưỡng cảnh báo.
- Giới hạn tốc độ gửi yêu cầu và thời gian chạy.
- Đặt thời gian lưu log phù hợp.
- Theo dõi tài nguyên và dọn dẹp sau thử nghiệm.

Cảnh báo ngân sách không tự động dừng toàn bộ tài nguyên. Công cụ tạo tải cần có giới hạn và điều kiện dừng riêng.

Không giả định toàn bộ thử nghiệm sẽ miễn phí hoặc nằm trong Free Tier.

### 10.2. Rủi ro và biện pháp

| Rủi ro | Biện pháp dự kiến |
|--------|------------------|
| Thông điệp trùng hoặc đến không đúng thứ tự | Idempotency, kiểm soát chuyển trạng thái và kiểm thử giao lại. |
| Ghi dữ liệu thành công nhưng phát thông điệp thất bại | Lựa chọn cơ chế tiếp nhận tin cậy; xem xét outbox và đối soát. |
| Tồn đọng tăng nhanh | Theo dõi tuổi thông điệp, giới hạn tải và điều chỉnh năng lực consumer trong ngân sách. |
| Retry kéo dài làm tăng chi phí | Phân loại lỗi, cấu hình ngưỡng và cô lập bằng DLQ. |
| Kết quả so sánh thiếu công bằng | Thống nhất nghiệp vụ, ghi rõ cấu hình và chạy lặp lại. |
| Khó xác định nguyên nhân lỗi | Log có cấu trúc, correlationId và quy trình tra cứu theo đơn hàng. |
| Lộ thông tin trong log hoặc mã nguồn | Sử dụng dữ liệu giả lập, hạn chế nội dung log và không lưu thông tin xác thực trong mã nguồn. |

## 11. Tài liệu tham khảo

- [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [Amazon CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [AWS Pricing Calculator](https://calculator.aws/)