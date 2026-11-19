---
title: "Triển khai"
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
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

Trả lời các câu hỏi như sau:

| Câu hỏi | Trả lời |
|---|---|
| Stack Name | `order-system` |
| AWS Region | `ap-southeast-1` |
| Confirm changes before deploy | `N` |
| Allow SAM CLI IAM role creation | `Y` |
| Disable rollback | `N` |
| Save arguments to samconfig.toml | `Y` |

Khi hoàn tất, SAM in ra `QueueUrl` và `DlqUrl`.

![Kết quả deploy](/images/5-Workshop/5.3-SQS-DLQ/deploy-result.png)

## Theo dõi từ lúc build đến khi có URL

SAM đọc template trong thư mục ứng dụng và tạo đầu ra build dưới `.aws-sam`. Đến bước deploy, CloudFormation mới nhận danh sách thay đổi. Trong phần này danh sách phải tương ứng với hai hàng đợi. Tên `order-system` và Region Singapore cần giữ nguyên cho các lần cập nhật ở mục 5.4 và 5.5. Nếu tạo stack mới ở mỗi chương, Lambda của chương sau có thể đọc một hàng đợi khác với hàng đợi đang kiểm thử.

Sau khi lệnh deploy kết thúc, dùng khối sau từ cùng profile để đọc kết quả trực tiếp. Hãy lấy URL từ output thực tế của tài khoản. Phần cuối URL nhận diện hàng đợi vật lý và có thể khác với ví dụ trong tài liệu.

```powershell
aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs}' --output json --no-cli-pager
```

## Phân biệt ba trạng thái triển khai

| Trạng thái | Cách đọc | Hành động |
|---|---|---|
| `CREATE_IN_PROGRESS` | AWS đang thực hiện change set | Chờ sự kiện cuối của stack |
| `CREATE_COMPLETE` | Stack đã tạo xong | Đọc hai output rồi chuyển sang kiểm tra redrive policy |
| `ROLLBACK_COMPLETE` khi tạo mới | CloudFormation đã hoàn tác lần tạo lỗi | Mở **Events**, tìm tài nguyên thất bại đầu tiên và xử lý nguyên nhân trước khi tạo lại |

Khi mẫu chưa thay đổi, một lần deploy lại có thể báo không có thay đổi. Kiểm tra trạng thái stack và Outputs đang có trước khi kết luận. Nếu chỉ thấy `Resource creation cancelled`, đọc ngược dòng thời gian đến sự kiện có thông báo cụ thể hơn, chẳng hạn thiếu quyền tạo queue.
