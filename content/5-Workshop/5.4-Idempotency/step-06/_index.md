---
title: "Troubleshooting"
weight: 6
chapter: false
pre: " <b> 5.4.6 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

- **Lambda does not run**: go to Console → Lambda → the `ProcessOrderFn` function → **Configuration → Triggers** tab, and check that the SQS trigger is *Enabled*.
- **DynamoDB permission error**: check the log, and make sure the `Policies` section of the template uses the correct `!Ref OrdersTable`.
- **`sam logs` shows nothing**: wait a few seconds and send the message again, as logs have a small delay.

## Narrow the search from queue to table

Start with a fresh order ID and a UTC send time. Confirm that SQS returned a message ID. Then look for an invocation of the intended function in the same interval. Visible messages with an inactive mapping point to the trigger. A fall in visible count with an increase in in-flight messages can mean processing is underway or a failed delivery is waiting for its visibility timeout.

```powershell
sam logs --stack-name order-system --name ProcessOrderFn --region ap-southeast-1 --start-time '15min ago'
```

Look for the ID or the first exception near its send time. For `Runtime.ImportModuleError` inspect the handler file, `CodeUri` and packaged dependencies. For denied PutItem access check the execution role and table ARN. A handled conditional write failure is expected for a duplicate. Keep the time alongside the observation because one stream can contain many orders.

## Verify a correction with a small case

Change one factor at a time. Rebuild and update the same stack after a code or template edit. Changing a log filter only requires reading the log again. To recheck new-order processing, use a fresh ID. To test the duplicate branch, intentionally reuse the ID. Record the original symptom, change made and observed outcome. These three lines document a reproducible repair.
