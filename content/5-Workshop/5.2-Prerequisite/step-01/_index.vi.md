---
title: "Quyền IAM"
weight: 1
chapter: false
pre: " <b> 5.2.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Trong PowerShell trên Windows, sao chép khối dưới và chạy một lần. Clipboard sẽ chứa **chỉ JSON của policy**. Mở **IAM → Policies → Create policy → JSON**, dán từ clipboard và đặt tên `OrderWorkshopDeployPolicy`.

```powershell
$policyJson = @'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WorkshopServices",
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "lambda:*",
        "apigateway:*",
        "sqs:*",
        "dynamodb:*",
        "states:*",
        "sns:*",
        "ses:*",
        "cloudwatch:*",
        "logs:*",
        "events:*",
        "s3:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "WorkshopIam",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:GetRole",
        "iam:PassRole",
        "iam:TagRole",
        "iam:UntagRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:GetRolePolicy",
        "iam:ListRolePolicies",
        "iam:ListAttachedRolePolicies"
      ],
      "Resource": "*"
    }
  ]
}
'@
$null = $policyJson | ConvertFrom-Json -ErrorAction Stop
Set-Clipboard -Value $policyJson
```

Policy này rộng để thuận tiện cho môi trường thử nghiệm. Với môi trường thật,
hãy giới hạn `Resource` theo từng tài nguyên cụ thể.

#### Tạo quyền (policy)
![Tạo IAM policy](/images/5-Workshop/5.2-Prerequisite/001.png)

<!-- workshop-detail:v1:begin -->
## Phân biệt quyền triển khai và quyền thực thi

Danh tính dùng AWS CLI cần quyền tạo hoặc cập nhật các tài nguyên trong template. Khi ứng dụng chạy, Lambda và Step Functions sử dụng các execution role riêng. Việc một người dùng mở được AWS Console không tự chứng minh các role của ứng dụng đã có đủ quyền.

| Thành phần | Ví dụ quyền cần có theo nhiệm vụ |
|---|---|
| Danh tính triển khai | CloudFormation và các dịch vụ được template sử dụng; tạo hoặc truyền role khi cần |
| Lambda xử lý | Đọc/xử lý SQS, thao tác DynamoDB và khởi động workflow theo thiết kế |
| Step Functions | Gọi SES, cập nhật DynamoDB và publish SNS theo các state |

Policy mẫu trong bài dành cho môi trường thử nghiệm và có phạm vi rộng. Khi áp dụng ngoài bài lab, cần giới hạn hành động và tài nguyên theo ARN thực tế. Các phần mở rộng như API/website cũng cần quyền tương ứng với template của chúng.

## Kiểm tra và xử lý AccessDenied

Trong IAM, mở user hoặc role đang dùng và xem mục Permissions. Khi gặp lỗi, ghi lại danh tính, action và resource được AWS nêu. Không suy đoán rằng mọi AccessDenied đều được giải quyết bằng cách thêm một policy rộng.

Nếu danh tính không có quyền gắn policy cho chính nó, cần người quản trị có thẩm quyền cấp quyền phù hợp. Lệnh STS thành công chỉ xác nhận danh tính, không chứng minh đã có đủ quyền triển khai.

<!-- workshop-detail:v1:end -->

## Chọn đúng danh tính khi gắn policy

Trong IAM, mở user dùng cho hồ sơ `order-workshop` và xem tab **Permissions**. Nếu nhóm hoặc role do quản trị viên cấp đã có các quyền cần thiết, ghi tên danh tính để dùng nhất quán. Nếu tài khoản lab dùng user riêng, tạo policy từ JSON ở đầu trang, gắn cho user đó, rồi mở lại danh sách quyền để kiểm tra đúng tên `OrderWorkshopDeployPolicy`. Việc xác minh email nhận cảnh báo chi phí có thể do người có quyền Billing thực hiện riêng.

Giả sử CloudFormation tạo bảng thành công nhưng gặp lỗi khi tạo hàm. Đọc sự kiện đầu tiên chứa tên action bị từ chối. Nếu action là `iam:PassRole`, danh tính triển khai đang cố giao role cho dịch vụ. Nếu Lambda đã chạy rồi mới ghi log lỗi `dynamodb:PutItem`, thành phần cần xem là execution role của Lambda và ARN bảng. Hai lỗi xảy ra ở các thời điểm khác nhau dù đều hiện từ “permission”.

## Ghi lại dữ kiện khi nhờ cấp quyền

Ghi thời điểm, Region, tên stack, action và tên tài nguyên từ thông báo lỗi. Quản trị viên có thể dựa vào đó để xem policy, permissions boundary hoặc quy tắc của tổ chức đang áp dụng. Khi quyền đã được cập nhật, thử lại đúng bước vừa lỗi trên cùng profile. Không đưa access key hoặc nội dung email khách vào bản ghi sự cố.
