---
title: "Xử lý đơn hàng với Lambda và SQS"
date: 2026-11-12
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

## Thử lại mà không tạo tác động trùng

Chào mọi người!

Khi tìm hiểu hệ thống xử lý đơn hàng theo kiến trúc event-driven, mình đặt ra một câu hỏi: **nếu một đơn hàng được gửi đến thành phần xử lý hai lần, hệ thống sẽ làm gì?**

Bài viết này tham khảo [bài về idempotency trên AWS Compute Blog](https://aws.amazon.com/blogs/compute/handling-lambda-functions-idempotency-with-aws-lambda-powertools/), đăng ngày **20/04/2022**, do **Jerome Van Der Linden và Dariusz Osiennik** viết. Bài giới thiệu cách sử dụng Powertools để hỗ trợ xử lý lặp an toàn.

Các tình huống và dữ liệu đơn hàng dưới đây do mình xây dựng để minh họa cho đề tài. Đây là bài tìm hiểu kỹ thuật, chưa phải báo cáo kết quả triển khai thực tế.

## 1. Một đơn hàng, hai lần xử lý

Giả sử hệ thống nhận đơn hàng `ORDER-DEMO-01`. Thành phần xử lý ghi nhận kết quả xác nhận đơn, nhưng gặp lỗi trước khi kết thúc lượt chạy.

Sau đó, thông điệp được xử lý lại.

Nếu chương trình luôn thực hiện toàn bộ nghiệp vụ mỗi khi nhận thông điệp, cùng một đơn có thể phát sinh hai bản ghi xác nhận hoặc hai thông báo.

Câu hỏi cần kiểm tra lúc này là: **nghiệp vụ đã thực hiện đến đâu trước khi lỗi xảy ra?**

Theo tài liệu AWS, tích hợp Lambda với SQS có thể xử lý một bản ghi nhiều lần. Lambda đọc thông điệp theo batch; khi xử lý batch thành công, các thông điệp được xóa khỏi hàng đợi. Vì vậy, consumer cần có khả năng tiếp nhận thông điệp lặp mà không tạo tác động ngoài ý muốn.

Tham khảo: [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html).

## 2. Idempotency giúp giải quyết điều gì?

Idempotency có thể hiểu là: thực hiện lại cùng một thao tác với cùng đầu vào không tạo thêm tác động ngoài kết quả đã có.

Powertools for AWS Lambda cung cấp tiện ích quản lý khóa, trạng thái và kết quả xử lý. Cơ chế này có thời hạn hiệu lực, cần được cấu hình phù hợp với khoảng thời gian có thể xảy ra retry.

Tham khảo: [Powertools — Idempotency](https://docs.aws.amazon.com/powertools/python/latest/utilities/idempotency/).

Trong ví dụ của mình, thông điệp có dạng:

```json
{
  "orderId": "ORDER-DEMO-01",
  "operation": "CONFIRM_ORDER",
  "correlationId": "TRACE-DEMO-01"
}
```

Mình đề xuất khóa nghiệp vụ:

```text
ORDER-DEMO-01#CONFIRM_ORDER
```

Cách đặt này thể hiện cả đơn hàng và thao tác cần thực hiện:

| Tình huống minh họa | Cách hiểu mong muốn |
|--------------------|-------------------|
| Xác nhận lại cùng đơn hàng | Kiểm tra kết quả của thao tác đã có. |
| Xác nhận một đơn hàng khác | Một công việc mới. |
| Hủy đơn hàng đã xác nhận | Một thao tác khác, phải kiểm tra điều kiện chuyển trạng thái. |

Đây là quy ước minh họa. Khi triển khai, cần xác định tính duy nhất của mã đơn hàng và phạm vi sử dụng khóa.

## 3. Không chỉ kiểm tra “đã có khóa hay chưa”

Một điểm đáng chú ý trong bài AWS là tình huống hai lượt xử lý đến gần như đồng thời, hoặc nghiệp vụ đã tạo tác động nhưng chương trình lỗi trước khi lưu kết quả chống trùng.

Do đó, một bước kiểm tra đơn giản rồi mới thực hiện công việc có thể chưa đủ.

Tham khảo: [Phân tích tình huống biên trong bài AWS Compute Blog](https://aws.amazon.com/blogs/compute/handling-lambda-functions-idempotency-with-aws-lambda-powertools/).

Với workshop của mình, tình huống cần thử là:

1. Lượt A bắt đầu xác nhận đơn hàng.
2. Lượt B nhận lại cùng công việc khi A chưa kết thúc.
3. A bị dừng giữa chừng.
4. Hệ thống nhận một lần thử lại tiếp theo.

Mình cần kiểm tra cả khả năng ngăn xử lý đồng thời lẫn khả năng tiếp tục công việc bị gián đoạn. Nếu chỉ ngăn mọi lần gọi sau, đơn hàng có thể bị mắc kẹt.

Nếu có tác động bên ngoài như gửi thông báo hoặc gọi dịch vụ thanh toán, phần đó cũng cần cơ chế chống trùng hay đối soát riêng. Một bản ghi trong DynamoDB không tự chứng minh toàn bộ chuỗi nghiệp vụ chỉ xảy ra đúng một lần.

## 4. Một thông điệp lỗi có khiến cả batch chạy lại?

Giả sử một batch có ba thông điệp:

| Thông điệp | Kết quả giả lập |
|------------|----------------|
| msg-demo-a | Thành công |
| msg-demo-b | Thất bại |
| msg-demo-c | Thành công |

Mặc định, lỗi xử lý batch có thể khiến cả các thông điệp đã thành công được nhận lại.

AWS hỗ trợ partial batch response để báo riêng những thông điệp thất bại. Với ví dụ trên, phản hồi có thể là:

```json
{
  "batchItemFailures": [
    {
      "itemIdentifier": "msg-demo-b"
    }
  ]
}
```

Cần bật `ReportBatchItemFailures` trên event source mapping và triển khai logic phản hồi tương ứng. Chỉ trả JSON mà chưa cấu hình mapping là chưa đủ.

Nếu handler ném exception ra ngoài, cả batch vẫn được xem là thất bại. Partial batch response cũng không thay thế cơ chế idempotency.

Tham khảo: [Handling errors for an SQS event source in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).

## 5. Dead-letter queue là nơi điều tra lỗi

Một thông điệp sai dữ liệu có thể tiếp tục thất bại dù được thử lại nhiều lần.

SQS cho phép chuyển thông điệp sang dead-letter queue thông qua cấu hình redrive policy. Tham số `maxReceiveCount` quy định ngưỡng số lần nhận trước khi thông điệp được chuyển sang DLQ.

DLQ hỗ trợ cô lập thông điệp để phân tích và phục hồi; bản thân hàng đợi này không sửa nguyên nhân gây lỗi.

Tham khảo: [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).

Trong workshop, mình dự kiến xử lý theo trình tự:

1. Tìm thông điệp lỗi và mã đơn hàng liên quan.
2. Tra cứu log để xác định bước thất bại.
3. Kiểm tra đơn hàng đã có tác động nghiệp vụ nào.
4. Sửa nguyên nhân và thử lại với một mẫu nhỏ.
5. Đối chiếu kết quả trước khi phục hồi các thông điệp còn lại.

## 6. Liên hệ với đề tài xử lý đơn hàng

Mô hình thử nghiệm dự kiến:

```text
API tiếp nhận
      |
      v
  Hàng đợi SQS -------> DLQ
      |
      v
 Lambda xử lý
      |
      v
DynamoDB: trạng thái đơn hàng và dữ liệu chống trùng
```

Log và metric sẽ được dùng để theo dõi các bước xử lý.

Đây là sơ đồ tập trung vào consumer. Cách bảo đảm việc tiếp nhận đơn hàng và đưa công việc vào hàng đợi cần được thiết kế riêng trong phần kiến trúc.

Mình đề xuất bộ thử nghiệm sau:

| Kịch bản | Điều cần kiểm tra |
|----------|------------------|
| Gửi lại cùng công việc | Có phát sinh thêm tác động nghiệp vụ không? |
| Hai lượt xử lý đồng thời | Có cùng vượt qua bước nhận quyền xử lý không? |
| Dừng hàm sau một bước nghiệp vụ | Lần thử lại có tiếp tục đúng trạng thái không? |
| Một thông điệp lỗi trong batch | Các thông điệp thành công có bị chạy lại không? |
| Phục hồi thông điệp từ DLQ | Kết quả đơn hàng có nhất quán sau phục hồi không? |

Khi so sánh với phương án đồng bộ, mình sẽ áp dụng cùng yêu cầu về tính đúng đắn cho cả hai phía. Đồng thời, thời gian API tiếp nhận đơn sẽ được đo riêng với thời gian hoàn tất nghiệp vụ.

Hiện bài viết chưa có số liệu để kết luận phương án nào nhanh hơn hoặc tiết kiệm hơn.

## 7. Góc nhìn của mình

Điều mình muốn kiểm chứng trong đề tài là một đơn hàng có thể được theo dõi và phục hồi đến trạng thái cuối cùng như thế nào, kể cả khi xảy ra lỗi giữa các bước.

Một bài lab phù hợp để bắt đầu là dùng vài đơn hàng giả lập, chủ động tạo lỗi và lưu lại bảng đối chiếu: số lần nhận thông điệp, số lần chạy hàm và số tác động nghiệp vụ thực sự.

Theo mình, bảng đối chiếu này sẽ giúp giải thích rõ hơn việc “xử lý nhiều lần” có dẫn đến “tạo kết quả nhiều lần” hay không.

Nếu đã từng xây dựng luồng tương tự, bạn kiểm tra tình huống lỗi giữa các bước bằng cách nào?

## Tài liệu tham khảo

1. [AWS Compute Blog — bài viết nền tảng về idempotency, ngày 20/04/2022](https://aws.amazon.com/blogs/compute/handling-lambda-functions-idempotency-with-aws-lambda-powertools/).
2. [AWS Lambda — Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html).
3. [Powertools for AWS Lambda — Idempotency](https://docs.aws.amazon.com/powertools/python/latest/utilities/idempotency/).
4. [AWS Lambda — Handling errors for an SQS event source](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).
5. [Amazon SQS — Using dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).