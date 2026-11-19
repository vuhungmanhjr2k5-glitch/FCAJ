---
title: "Tạo file template.yaml"
weight: 1
chapter: false
pre: " <b> 5.3.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

## Ghi toàn bộ `template.yaml`

Từ PowerShell đang mở ở thư mục ứng dụng `order-system`, chạy các khối theo thứ tự. Khối ghi tệp dùng UTF-8 không BOM. Nếu đã chỉnh riêng tệp cùng tên, lưu thay đổi của bạn trước khi thay nội dung.

```powershell
$content = @'
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Order workshop - SQS va DLQ

Resources:
  OrdersDLQ:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600   # giu tin nhan loi 14 ngay

  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 60
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrdersDLQ.Arn
        maxReceiveCount: 3

Outputs:
  QueueUrl:
    Value: !Ref OrdersQueue
  DlqUrl:
    Value: !Ref OrdersDLQ
'@
$path = Join-Path $PWD.Path 'template.yaml'
[IO.File]::WriteAllText($path, ($content + [Environment]::NewLine), [Text.UTF8Encoding]::new($false))
```

## Đọc template từ hàng đợi lỗi đến hàng đợi chính

Đọc tài nguyên `OrdersDLQ` trước. Hàng này đặt thời gian lưu 1.209.600 giây, tương đương 14 ngày. Với SQS Standard, tuổi dùng để tính thời hạn lưu vẫn gắn với thời điểm gửi ban đầu khi thông điệp được chuyển sang DLQ. Thời gian lưu dài tạo thêm cơ hội điều tra một đơn lỗi. Việc chuyển hàng đợi không khởi động lại toàn bộ thời hạn đó.

Ở `OrdersQueue`, thuộc tính `deadLetterTargetArn` lấy ARN của chính tài nguyên `OrdersDLQ` trong template. Hai hàng đợi có thể mang tên vật lý khác nhau giữa các tài khoản. Tham chiếu theo logical ID giúp liên kết vẫn đúng sau mỗi lần tạo stack. Outputs trả về URL vì các thao tác gửi, nhận và đọc thuộc tính của SQS dùng URL làm đầu vào.

## Theo dõi một lần giao lại

Giả sử consumer nhận một tin lúc 10:00:00. Với `VisibilityTimeout` bằng 60 giây, consumer khác thông thường sẽ chưa thấy tin đó trong thời gian ẩn. Nếu xử lý xong và xóa tin, lần giao kết thúc. Nếu không xóa, tin có thể được nhận lại khi hết thời gian ẩn. Mỗi lần **nhận thực tế** làm tăng receive count. Chỉ để tin nằm yên trong queue không làm nó tự tiến dần tới DLQ.

## Đọc lại tệp trước khi build

```powershell
Get-Item -LiteralPath .\template.yaml -ErrorAction Stop | Select-Object FullName, Length
Select-String -LiteralPath .\template.yaml -Pattern '^  (OrdersQueue|OrdersDLQ|QueueUrl|DlqUrl):'
```

Kết quả cần có hai định danh tài nguyên và hai đầu ra, mỗi tên xuất hiện đúng cấp thụt lề trong tệp. Nếu template không có trong thư mục hiện tại, quay lại thư mục ứng dụng ở mục 5.2.6. Bước này có thể kiểm tra hoàn toàn ở máy, trước khi phát sinh tài nguyên SQS.
