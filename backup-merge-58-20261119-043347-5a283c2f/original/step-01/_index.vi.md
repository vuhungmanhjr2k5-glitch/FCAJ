---
title: "Xóa stack chính"
weight: 1
chapter: false
pre: " <b> 5.8.1 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

```powershell
sam delete --stack-name order-system --region ap-southeast-1
```

Trả lời `y` cho các câu hỏi xác nhận. Lệnh này xóa stack CloudFormation cùng
SQS, DLQ, DynamoDB, Lambda, Step Functions, SNS, CloudWatch alarm, dashboard và
HTTP API.

Kiểm tra stack đã biến mất:

```powershell
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE DELETE_FAILED --query "StackSummaries[].StackName" --output text
```

![Kết quả xóa stack](/images/5-Workshop/5.8-Cleanup/delete-stack.png)
