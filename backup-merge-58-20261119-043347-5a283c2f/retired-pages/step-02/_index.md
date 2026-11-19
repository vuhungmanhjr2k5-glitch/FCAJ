---
title: "Delete the Lambda log groups"
weight: 2
chapter: false
pre: " <b> 5.8.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Lambda creates its log groups automatically, so they are not deleted with the
stack:

```powershell
$groups = aws logs describe-log-groups --log-group-name-prefix /aws/lambda/order-system --query "logGroups[].logGroupName" --output text
$groups -split "\s+" | Where-Object { $_ } | ForEach-Object { aws logs delete-log-group --log-group-name $_ }
```
