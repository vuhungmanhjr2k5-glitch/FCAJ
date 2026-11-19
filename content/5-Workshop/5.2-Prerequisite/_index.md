---
title: "Preparation (IAM, Region, Tools)"
date: "2026-10-03"
weight: 2
chapter: false
pre: " <b> 5.2 </b> "
---

<!-- workshop-split:v1 -->

## Prerequisites

- An AWS account for experimentation (do not use a production account)
- A real email address to receive test emails
- A Windows computer with PowerShell and Git installed
- Region used throughout the workshop: **Asia Pacific (Singapore) `ap-southeast-1`**

<!-- workshop-detail:v1:begin -->
## Purpose of preparation

Preparation ensures that later deployment steps use the intended identity, Region and tools. This helps distinguish architecture problems from missing permissions, unverified email identities or an incomplete local setup.

## Consistent settings

| Setting | Workshop convention |
|---|---|
| Region | Singapore, ap-southeast-1 |
| Account | An authorized account for the lab |
| CLI profile | Use the configured profile consistently, for example order-workshop |
| Email | Sender and recipient meet SES requirements in the selected Region |
| Project folders | Keep AWS application code separate from Hugo documentation |

Changing the Console Region does not change the CLI Region. Specify --region ap-southeast-1 when checking resources. IAM is global; resources such as SQS, Lambda, DynamoDB and Step Functions, as well as SES sandbox status, are Region-specific.

## Ready to continue

Version checks return successfully, STS identifies the intended account, test email is ready, deployment permissions are available and code/evidence have a known location. Preparation screenshots can come from a single check; creating extra resources just for illustrations is unnecessary.
<!-- workshop-detail:v1:end -->

## Workshop steps

Open a page below or use the sidebar. Follow the steps in order.

- [5.2.1 IAM permissions](step-01/)
- [5.2.2 Install the tools](step-02/)
- [5.2.3 Connect your computer to your AWS account](step-03/)
- [5.2.4 Verify your email in Amazon SES](step-04/)
- [5.2.5 Set up a cost alert](step-05/)
- [5.2.6 Create the project folder](step-06/)
