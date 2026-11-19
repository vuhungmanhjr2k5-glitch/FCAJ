---
title: "Preparation (IAM, Region, Tools)"
date: "2026-10-03"
weight: 2
chapter: false
pre: " <b> 5.2 </b> "
---

## Prerequisites

- An AWS account for experimentation (do not use a production account)
- A real email address to receive test emails
- A Windows computer with PowerShell and Git installed
- Region used throughout the workshop: **Asia Pacific (Singapore) `ap-southeast-1`**

#### 1. IAM permissions

Create the following policy and attach it to the user you use to deploy the workshop
(IAM → Policies → Create policy → *JSON* tab → paste the content → name it `OrderWorkshopDeployPolicy`).

```json

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

```

> This policy is broad for the convenience of the test environment. For a real environment,
> restrict the `Resource` to each specific resource.


![Create IAM policy](/images/5-Workshop/5.2-Prerequisite/001.png)

#### 2. Install the tools

Open PowerShell and run:
```powershell
winget install -e --id OpenJS.NodeJS.LTS
winget install -e --id Amazon.AWSCLI
winget install -e --id Amazon.SAM-CLI
```


Close and reopen PowerShell, then verify:

```powershell
node -v
aws --version
sam --version
```
#### 3. Connect your computer to your AWS account

1. Go to IAM → Users → select the user → *Security credentials* → *Create access key*
   (use case: Command Line Interface).
2. Configure the profile:
```powershell
aws configure --profile order-workshop
$env:AWS_PROFILE = "order-workshop"
aws sts get-caller-identity
```

Enter the Access Key, Secret Key, region `ap-southeast-1`, and output `json`.
The result of the last command should display your account information.

![Verify identity](/images/5-Workshop/5.2-Prerequisite/002.png)

>Do not push the Access Key to GitHub. Delete or deactivate the key after you finish the workshop.


#### 4. Verify your email in Amazon SES

A new SES account is in *sandbox* mode: it can only send emails *from and to* verified addresses.

1. Go to *Amazon SES* → *Identities* → *Create identity*.
2. Choose *Email address*, enter your email, and select *Create identity*.
3. Open the verification email from AWS and click the link.
4. The identity status changes to *Verified*.

![SES verified](/images/5-Workshop/5.2-Prerequisite/003.png)

#### 5. Set up a cost alert

1. Go to *Billing and Cost Management* → *Budgets* → *Create budget*.
2. Choose *Zero spend budget* or *Monthly cost budget* (for example, 5 USD).
3. Enter the email to receive alerts.

![Create budget](/images/5-Workshop/5.2-Prerequisite/004.png)

#### 6. Create the project folder
```powershell
mkdir order-system
cd order-system
mkdir src
```
In the following sections, we will add`template.yaml` and the Lambda source code to this folder.