---
title: "Dọn dẹp tài nguyên"
weight: 8
chapter: false
pre: " <b> 5.8 </b> "
---

<!-- workshop-merged-58:v1 -->

## Mục tiêu

Xóa toàn bộ tài nguyên đã tạo để không phát sinh chi phí, đặc biệt là hai API
công khai ở mục 5.7.

## Trước khi xóa

Lưu kết quả ở mục 5.7 và kiểm tra đúng profile, Region, tên stack. Nếu đã triển khai stack giám sát riêng ở 5.6.3, dọn stack đó trước để tránh giữ alarm trỏ vào các tài nguyên sắp xóa. Chỉ thực hiện khối dưới khi `order-monitoring` là stack dành riêng cho bài thực hành của bạn.

```powershell
$monitorStack = 'order-monitoring'
aws cloudformation describe-stacks --stack-name $monitorStack --region ap-southeast-1 --query 'Stacks[0].{Stack:StackName,Status:StackStatus}' --output table --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Check the monitoring stack name and Region.' }
if ((Read-Host "Type $monitorStack to remove this lab monitoring stack") -ceq $monitorStack) {
    aws cloudformation delete-stack --stack-name $monitorStack --region ap-southeast-1
    if ($LASTEXITCODE -ne 0) { throw 'Deletion request failed.' }
    aws cloudformation wait stack-delete-complete --stack-name $monitorStack --region ap-southeast-1
    if ($LASTEXITCODE -ne 0) { throw 'Inspect stack events before continuing.' }
}
```

## 1. Xóa stack chính

```powershell
sam delete --stack-name order-system --region ap-southeast-1
```

Trả lời `y` cho các câu hỏi xác nhận. Lệnh này xóa stack CloudFormation cùng
SQS, DLQ, DynamoDB, Lambda, Step Functions, SNS, các tài nguyên khác thuộc stack và
HTTP API.

Kiểm tra lần triển khai mới nhất của stack đã có trạng thái `DELETE_COMPLETE`. CloudFormation có thể giữ lịch sử những lần tạo và xóa trước đây.

```powershell
aws cloudformation list-stacks --region ap-southeast-1 --query "StackSummaries[?StackName=='order-system'].[StackName,StackStatus,CreationTime]" --output table --no-cli-pager
```

![Kết quả xóa stack](/images/5-Workshop/5.8-Cleanup/delete-stack.png)

## 2. Xóa log group của Lambda

Log group do Lambda tự tạo nên không bị xóa cùng stack:

```powershell
$groups = aws logs describe-log-groups --region ap-southeast-1 --log-group-name-prefix '/aws/lambda/order-system-' --query 'logGroups[].logGroupName' --output json --no-cli-pager
if ($LASTEXITCODE -ne 0) { throw 'Cannot list log groups.' }
foreach ($name in ($groups | Out-String | ConvertFrom-Json)) {
    if ((Read-Host "Type DELETE to remove the lab log group $name") -ceq 'DELETE') {
        aws logs delete-log-group --region ap-southeast-1 --log-group-name $name
        if ($LASTEXITCODE -ne 0) { throw "Could not delete $name." }
    }
}
```

## 3. Xóa stack quản lý của SAM (tùy chọn)

Chỉ dọn tài nguyên SAM dùng riêng cho lab. SAM tạo thêm stack `aws-sam-cli-managed-default` kèm một S3 bucket chứa mã
triển khai. Chi phí rất nhỏ nhưng bạn có thể dọn: vào **S3**, mở bucket có tên
bắt đầu bằng `aws-sam-cli-managed-default`, chọn **Empty**, sau đó vào
**CloudFormation** xóa stack `aws-sam-cli-managed-default`.

## 4. Xóa email đã xác minh trong SES (tùy chọn)

Vào **Amazon SES** → **Identities** → chọn email → **Delete**.

## 5. Thu hồi quyền truy cập

Làm bước này **sau cùng**, vì sau khi xóa khóa thì CLI không dùng được nữa:

1. Vào **IAM** → **Users** → chọn user → **Security credentials** → **Deactivate** rồi **Delete** access key.
2. Gỡ policy `OrderWorkshopDeployPolicy` khỏi user, rồi xóa policy nếu không dùng nữa.
3. Xóa profile trên máy: mở file `%USERPROFILE%\.aws\credentials` và `config`, xóa phần `order-workshop`.
4. Xóa biến môi trường trong PowerShell:

```powershell
Remove-Item Env:API_KEY, Env:CUSTOMER_EMAIL -ErrorAction SilentlyContinue
```

## 6. Kiểm tra lần cuối

1. Vào Console của **Lambda**, **SQS**, **DynamoDB**, **Step Functions**, **API Gateway**, **CloudWatch** (Alarms, Dashboards): không còn tài nguyên `order-system` nào.
2. Vào **Billing and Cost Management** → **Cost Explorer**, kiểm tra lại sau 1 đến 2 ngày để chắc chắn không còn phí mới từ các tài nguyên đã xóa.
3. Giữ lại ngân sách trong **AWS Budgets** nếu bạn còn dùng tài khoản này.

![Kiểm tra chi phí](/images/5-Workshop/5.8-Cleanup/cost-check.png)

## 7. Checklist dọn dẹp

| Việc | Xong |
|---|---|
| Xóa stack `order-system` | ☑ |
| Xóa log group Lambda | ☑ |
| Xóa stack và bucket của SAM (tùy chọn) | ☑ |
| Xóa identity SES (tùy chọn) | ☑ |
| Xóa access key và policy IAM | ☑ |
| Xóa profile và biến môi trường trên máy | ☑ |
| Kiểm tra Cost Explorer sau 1 đến 2 ngày | ☑ |
