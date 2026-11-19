---
title: "Thu hồi quyền truy cập"
weight: 5
chapter: false
pre: " <b> 5.8.5 </b> "
---

<!-- workshop-split:v1 -->

[← Mục lục phần này](../)

Làm bước này **sau cùng**, vì sau khi xóa khóa thì CLI không dùng được nữa:

1. Vào **IAM** → **Users** → chọn user → **Security credentials** → **Deactivate** rồi **Delete** access key.
2. Gỡ policy `OrderWorkshopDeployPolicy` khỏi user, rồi xóa policy nếu không dùng nữa.
3. Xóa profile trên máy: mở file `%USERPROFILE%\.aws\credentials` và `config`, xóa phần `order-workshop`.
4. Xóa biến môi trường trong PowerShell:

```powershell
Remove-Item Env:API_KEY, Env:CUSTOMER_EMAIL -ErrorAction SilentlyContinue
```
