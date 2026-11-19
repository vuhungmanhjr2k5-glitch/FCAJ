---
title: "Verify your email in Amazon SES"
weight: 4
chapter: false
pre: " <b> 5.2.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

In the SES sandbox, use a verified sender identity. Recipients can be verified addresses or SES mailbox simulator addresses.

1. Go to *Amazon SES* → *Identities* → *Create identity*.
2. Choose *Email address*, enter your email, and select *Create identity*.
3. Open the verification email from AWS and click the link.
4. The identity status changes to *Verified*.

![SES verified](/images/5-Workshop/5.2-Prerequisite/003.png)

<!-- workshop-detail:v1:begin -->
## Why email preparation matters

SES sends customer confirmations. Sandbox status and identity verification apply per Region. An address verified in another Region is not sufficient for a workflow running in Singapore.

In the sandbox, ordinary sender and recipient addresses must meet verification requirements. SES mailbox simulator addresses are an exception for testing. Production access removes the requirement to verify recipients individually, while sender identities still need to be valid.

## Check account status

```powershell
aws sesv2 get-account --region ap-southeast-1 --query '{Production:ProductionAccessEnabled,Sending:SendingEnabled}' --output json --no-cli-pager
```

Production=false indicates sandbox status. Production=true indicates production access; Sending reports whether account-level sending is enabled. This command only reads information: it neither sends mail nor requests production access.

## Prepare a test recipient

Open SES in Singapore and inspect Verified identities. If a new address is needed, create its email identity and have the mailbox owner follow the AWS verification link. Wait for Verified before submitting a new test order.

SNS subscription confirmation and SES identity verification are separate actions. Confirming an SNS subscription does not verify the address in SES. success@simulator.amazonses.com supports successful send-task tests but does not provide a recipient inbox for a delivered-email screenshot.

The verification screenshot below does not display a Region or the account's production-access status. Check those separately before drawing conclusions about sending to other recipients.
<!-- workshop-detail:v1:end -->

<!-- workshop-proof:v1:begin -->
## Email verification evidence

{{< workshop-proof file="Verified.png" alt="Verified email identity in Amazon SES" caption="Amazon SES lists vuhungmanhjr2k5@gmail.com as an Email address identity with Verified status." >}}

This confirms verification in the displayed identity list. The screenshot does not show the Region or production-access status. Check ap-southeast-1 and sandbox status separately before sending to another recipient. Verified does not establish successful submission or delivery of an order confirmation.
<!-- workshop-proof:v1:end -->

## Check the address that will send mail

Enter the email identity verified in your lab account. The following command reads its Singapore identity record. Look for `VerifiedForSendingStatus` set to `true` before supplying this address as the deployment parameter `SenderEmail`. The example uses an individual email identity as created above.

```powershell
$sender = Read-Host 'SES sender email'
aws sesv2 get-email-identity --email-identity $sender --region ap-southeast-1 --query '{Type:IdentityType,VerifiedForSendingStatus:VerifiedForSendingStatus}' --output table --no-cli-pager
```

For `NotFoundException`, check the Region and the spelling of the address. For a pending identity, open the matching mailbox and follow the current AWS verification link. Keep a note of the sender and recipient roles used in your test so the deployment prompts are easy to complete.

## Select the recipient for the question you are testing

Use the success simulator when measuring the API and checking whether SES accepted a task. To observe actual mailbox delivery, use a test inbox you control and locate the subject containing the order ID. For an administrative alert, inspect the SNS subscription and the topic that sent the notification. These checks answer different questions even when the same email address serves two roles.
