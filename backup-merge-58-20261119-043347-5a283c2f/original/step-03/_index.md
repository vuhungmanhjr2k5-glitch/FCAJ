---
title: "Delete the SAM managed stack (optional)"
weight: 3
chapter: false
pre: " <b> 5.8.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

SAM also creates the `aws-sam-cli-managed-default` stack along with an S3
bucket that holds the deployment code. The cost is very small, but you can
clean it up: go to **S3**, open the bucket whose name starts with
`aws-sam-cli-managed-default`, choose **Empty**, then go to **CloudFormation**
and delete the `aws-sam-cli-managed-default` stack.
