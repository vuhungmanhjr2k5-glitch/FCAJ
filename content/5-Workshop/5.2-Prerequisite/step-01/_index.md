---
title: "IAM permissions"
weight: 1
chapter: false
pre: " <b> 5.2.1 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Run the block in Windows PowerShell to place **only the policy JSON** on the clipboard. Open **IAM → Policies → Create policy → JSON**, paste and name it `OrderWorkshopDeployPolicy`.

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

> This policy is broad for the convenience of the test environment. For a real environment,
> restrict the `Resource` to each specific resource.


![Create IAM policy](/images/5-Workshop/5.2-Prerequisite/001.png)

<!-- workshop-detail:v1:begin -->
## Deployment permissions versus runtime permissions

The CLI identity needs permission to create or update resources in the template. At runtime, Lambda and Step Functions use their own execution roles. Access to the Console does not establish that the application roles have every required permission.

| Component | Example responsibilities requiring permission |
|---|---|
| Deployment identity | CloudFormation and template services; create/pass roles where required |
| Worker Lambda | Process SQS messages, access DynamoDB and start the workflow |
| Step Functions | Call SES, update DynamoDB and publish to SNS as defined by its states |

The example policy is broad for a lab environment. For other uses, restrict actions and resources to the actual ARNs. API and storefront extensions require permissions matching their own templates.

## Investigating AccessDenied

Open the active user or role in IAM and inspect Permissions. For a failure, record the identity, action and resource in the AWS error. Do not assume every AccessDenied can be solved by attaching a broad policy.

An identity without permission to attach policies to itself needs an authorized administrator to grant appropriate access. A successful STS response establishes identity, not complete deployment permissions.

Reuse an existing Permissions screenshot where appropriate. There is no need to create another user or policy just to obtain an illustration.
<!-- workshop-detail:v1:end -->

## Attach the policy to the identity used for deployment

In IAM, open the user used by the `order-workshop` profile and inspect **Permissions**. If an administrator has provided a group or role with the required access, record that identity for use throughout the lab. For a dedicated lab user, create the policy from the JSON at the top, attach it to that user and check that `OrderWorkshopDeployPolicy` appears in the permissions list. A billing administrator may handle budget setup separately.

Suppose CloudFormation creates the table and then fails while creating the function. Read the first event that identifies a denied action. A denial of `iam:PassRole` concerns the identity handing a role to a service. A denial of `dynamodb:PutItem` in a worker invocation points to the Lambda execution role and the table ARN. The point in the run tells you which identity to inspect.

## Record a useful access request

Capture the time, Region, stack name, denied action and resource from the error. This gives an administrator enough context to check the policy, permissions boundary and relevant organization controls. Once access is updated, retry the same operation with the same profile. Leave access keys and customer email content out of the incident record.
