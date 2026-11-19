---
title: "Connect your computer to your AWS account"
weight: 3
chapter: false
pre: " <b> 5.2.3 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

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

<!-- workshop-detail:v1:begin -->
## Verify the actual CLI identity and Region

The Console and CLI may use different identities. Check which account the CLI actually accesses before deployment. If your organization provides IAM Identity Center access, use its profile instructions instead of creating an extra access key just to follow an example.

For an already configured order-workshop profile:

```powershell
aws sts get-caller-identity --profile order-workshop --region ap-southeast-1 --query '{Account:Account,Arn:Arn}' --output table --no-cli-pager
aws configure get region --profile order-workshop
```

The first command should return the intended Account and Arn. The second reads the Region stored in the profile. Empty output means that value is not stored there; it does not confirm the Console Region. Specifying --region ap-southeast-1 makes the intended Region explicit.

## Avoid deploying to the wrong environment

Use the same profile throughout. Recheck AWS_PROFILE after opening another terminal, since a variable set in the previous session may not persist. Refresh an expired session for ExpiredToken errors. If STS succeeds but a service returns AccessDenied, inspect that service's permissions.

Do not include access keys, secret keys or session tokens in source code or screenshots. Existing working access can be checked without generating another key.

An identity screenshot should show the STS result and configured Region, with personal account information redacted as needed. SES identity verification is not a substitute for this CLI identity check.
<!-- workshop-detail:v1:end -->

## Resume in a new PowerShell session

Profile settings persist on disk. Environment variables set in a PowerShell window apply to that session and its child processes. At the start of a new session for sections 5.3–5.7, run the following block. Replace the profile name on the first line if your class provided a different one.

```powershell
$env:AWS_PROFILE = 'order-workshop'
$env:AWS_REGION = 'ap-southeast-1'
$env:AWS_DEFAULT_REGION = $env:AWS_REGION
aws sts get-caller-identity --output table --no-cli-pager
```

Compare `Account` with your lab account. An assumed-role ARN includes a session name at the end. If an IAM Identity Center session has expired, sign in again with the same profile and repeat the check. Choosing Singapore in the browser changes the Console view. The environment settings also direct CLI commands and the later Node.js benchmark to Singapore.

## Investigate an apparently missing stack

Check the active profile, the Region used for the call and the recorded stack name, in that order. Stack names can recur across Regions. A successful STS call and an empty SQS list can both be correct if the resources were created elsewhere. Start deployment when both the account and Region agree with your session notes.
