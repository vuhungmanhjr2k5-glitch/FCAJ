---
title: "Troubleshooting"
weight: 6
chapter: false
pre: " <b> 5.5.6 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

- **Not receiving the order confirmation email**: check that both `SenderEmail` and the customer email are *Verified* in SES, and check your spam folder.
- **Not receiving the alert**: check whether you clicked *Confirm subscription* in SNS (Console → SNS → Subscriptions, the status must be *Confirmed*).
- **No execution at all**: check the `ProcessOrderFn` logs; if there is a permissions error, review the `Policies` section in the template.
- **Execution reports an `AccessDenied` error at SES**: check that the state machine's role has `ses:SendEmail`.

## Trace the symptom to the responsible service

For a missing confirmation, start with the customer's order ID. A missing item leads back to the queue and worker. A `RECEIVED` item leads to the execution name. Once an execution is found, inspect the final state reached and the first failure. This sequence avoids changing SES verification when the worker has not yet started the state machine.

If the SES task reports access denied, inspect the state machine role and the allowed action. If SES reports an unverified identity, check the sender, test recipient and Region. For rate limiting after a burst, read retry history before submitting again manually. A manual resend adds another request to the experiment and can obscure the original cause.

## Investigate FAILED with no administrative email

Find `NotifyAdminFailure` in execution history. If it was never reached, inspect the preceding table update. For a denied publish, identify the role and topic ARN from the error. For a successful publish, check the confirmed subscription, inbox and spam folder of the intended administrative address. Record the last confirmed step before attributing the incident.

After the correction, use a fresh test order. Keep the old execution as a reference for comparison. The old ID, new ID and single change between runs document the repair.
