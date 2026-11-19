---
title: "Triển khai"
weight: 3
chapter: false
pre: " <b> 5.5.3 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

```powershell
sam validate --region ap-southeast-1
if ($LASTEXITCODE -ne 0) { throw 'Template validation failed.' }
sam build
if ($LASTEXITCODE -ne 0) { throw 'Build failed.' }
sam deploy --guided --region ap-southeast-1 --no-fail-on-empty-changeset
if ($LASTEXITCODE -ne 0) { throw 'Inspect the CloudFormation deployment events.' }
```

Giữ các câu trả lời như lần trước. Khi được hỏi hai tham số, nhập:

| Tham số | Giá trị |
|---|---|
| SenderEmail | Email đã xác minh ở mục 5.2 |
| AdminEmail | Email nhận cảnh báo (có thể là cùng email trên) |

![Kết quả deploy](/images/5-Workshop/5.5-Orchestration/deploy-result.png)

**Xác nhận đăng ký SNS:** AWS gửi một email tiêu đề *AWS Notification -
Subscription Confirmation* đến `AdminEmail`. Mở email và bấm **Confirm
subscription**, nếu không bạn sẽ không nhận được cảnh báo.

![Xác nhận SNS](/images/5-Workshop/5.5-Orchestration/sns-confirm.png)

## Kiểm tra hai loại xác nhận qua email

Trước khi build, mở lại hồ sơ `order-workshop` và vào thư mục có template điều phối. So sánh `SenderEmail` với địa chỉ SES đã xác minh ở Singapore. `AdminEmail` phải là hộp thư có người theo dõi trong lúc thực hành. Hai giá trị có thể giống nhau, nhưng thao tác xác minh identity trong SES và thao tác xác nhận subscription SNS diễn ra ở hai dịch vụ khác nhau.

Khi stack hoàn tất, mở email đăng ký SNS đúng topic rồi xác nhận một lần. Có thể kiểm tra trạng thái bằng block sau. Khối lệnh lấy ARN topic từ logical ID của stack. Một subscription email sẵn sàng sẽ hiển thị ARN thực ở cột cuối. `PendingConfirmation` nghĩa là nguồn có thể publish nhưng đăng ký email còn chờ người nhận xác nhận.

```powershell
$topic = aws cloudformation describe-stack-resource --stack-name order-system --logical-resource-id OrderTopic --region ap-southeast-1 --query 'StackResourceDetail.PhysicalResourceId' --output text --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'OrderTopic was not found.' }
aws sns list-subscriptions-by-topic --topic-arn $topic --region ap-southeast-1 --query 'Subscriptions[].{Channel:Protocol,Subscription:SubscriptionArn}' --output table --no-cli-pager
```

## Kiểm tra điều kiện sang phần gửi đơn

Mở Outputs và kiểm tra `StateMachineArn` cùng `TableName` vẫn thuộc stack đang dùng. Tìm tên hàm từ Resources, mở phần cấu hình để kiểm tra hai biến `STATE_MACHINE_ARN` và `TABLE_NAME`. Chúng phải trỏ đến state machine và bảng của cùng lần triển khai. Nếu deploy báo tham số còn thiếu, mở lại guided deploy của đúng template và điền cả hai email. Khi mọi liên kết đã khớp, dùng **một** đơn hợp lệ trước khi thử trùng hoặc cố ý gây lỗi.

### Nếu dùng lại hộp thư từ bước chuẩn bị

Một hộp thư có thể chứa hai thư xác nhận từ AWS. Thư SES liên quan đến quyền dùng địa chỉ làm identity ở Region đã chọn. Thư SNS ghi tên topic và có hành động xác nhận đăng ký nhận notification. Hãy kiểm tra tên topic có khớp `OrderTopic` vừa triển khai rồi mới xác nhận. Nếu stack tạo lại topic với ARN khác, subscription hợp lệ của topic cũ không tự chuyển sang topic mới.
