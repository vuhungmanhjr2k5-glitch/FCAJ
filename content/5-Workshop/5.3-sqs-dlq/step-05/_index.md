---
title: "Verify the result"
weight: 5
chapter: false
pre: " <b> 5.3.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

- There are two queues, and the main queue points to the DLQ with `maxReceiveCount = 3`.
- A message received more than 3 times without being deleted ends up in the DLQ.

## Combine a configuration result with a delivery result

Use two rows for acceptance. The configuration row contains the redrive destination, actual DLQ ARN and `maxReceiveCount`. The behavior row identifies the test body, at least one matching source receive and the destination where you found the same body. This lets a reader distinguish a configured failure path from a message actually observed on that path.

| Observation | Interpretation | Next check |
|---|---|---|
| Destination matches and the test body is in the DLQ | Configuration and transfer were observed for this run | Record the time and finish the test |
| DLQ count increased but the body is unidentified | A message is waiting | Look for the run marker |
| A source message is in flight | A visibility interval is active | Recheck after it expires |
| No successful source receives | The experiment has not reached the receive threshold | Verify the source URL and active consumers |

## Hand the queues over to the worker

Record whether any test message remains in the source. Once the trigger is enabled, Lambda can immediately receive an earlier plain-text body. A parse error just after deployment may belong to that item. Its body and send time distinguish it from a new order in 5.4.4. A message already in the DLQ remains subject to its own handling and retention.
