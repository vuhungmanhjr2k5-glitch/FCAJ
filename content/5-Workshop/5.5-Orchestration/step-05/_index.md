---
title: "Verify the results"
weight: 5
chapter: false
pre: " <b> 5.5.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

| Case | Expected result |
|---|---|
| Valid order | Email sent to the customer, execution Succeeded, status `DONE` |
| Duplicate submission | No second email, log reports a duplicate |
| Email cannot be sent | 3 retries, execution Failed, status `FAILED`, admin receives an alert |

## Resolve an order ID to a conclusion

Take an execution name from 5.5.4, read its final status and look up the `orderId` item. A full table scan can mix current observations with an older run. Looking up the same key in both services shows whether the workflow ended and what the application records. For a real recipient add the matching email subject and arrival time.

For duplicate processing, note how many times you resubmitted and whether the first execution had reached `DONE`. A resubmission during a running execution can follow a different branch from a resubmission afterward. Both can result in one execution with different worker logs. Report the timing actually tested.

## Interpret an unfinished observation

| Observation | What is established | Next check |
|---|---|---|
| `RECEIVED`, no identified execution | The initial item exists | Worker log and StartExecution permissions |
| Running execution | It may be waiting for a retry | Last event and scheduled attempt |
| `FAILED`, no email yet | The table update completed | NotifyAdminFailure outcome and subscription |
| `DONE` with the simulator | The workflow used its success path | SES event and recipient type |

A three-row table can document the order IDs, workflow outcomes, table states and mailbox observations. Where one source is still missing, record the check time and keep the conclusion open.

### Read the named execution

The Standard workflow in the template uses the order ID as its execution name. Enter the trial ID and read its status, start and end times. For ExecutionDoesNotExist, check the name and stack before concluding that the worker never started it.

```powershell
$orderId = Read-Host 'Order ID'
$sm = aws cloudformation describe-stacks --stack-name order-system --region ap-southeast-1 --query "Stacks[0].Outputs[?OutputKey=='StateMachineArn'].OutputValue | [0]" --output text --no-cli-pager
if ($LASTEXITCODE -ne 0 -or $sm -eq 'None') { throw 'StateMachineArn unavailable.' }
$executionArn = ($sm -replace ':stateMachine:', ':execution:') + ':' + $orderId
aws stepfunctions describe-execution --execution-arn $executionArn --region ap-southeast-1 --query '{State:status,Start:startDate,End:stopDate}' --output table --no-cli-pager
```
