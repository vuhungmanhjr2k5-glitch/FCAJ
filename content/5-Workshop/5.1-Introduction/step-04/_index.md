---
title: "Duration and cost"
weight: 4
chapter: false
pre: " <b> 5.1.4 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

<!-- workshop-detail:v1:begin -->
## Planning the work

The following ranges are planning estimates, not measured completion times. Actual duration depends on installed tools, available permissions and any environment issues that need troubleshooting.

| Activity | Suggested time |
|---|---|
| Read the architecture and check tools/identity | 30–45 minutes |
| Deploy queues, duplicate controls and orchestration | 60–90 minutes |
| Test failures, dashboards and alarms | 30–60 minutes |
| Collect evidence and clean up | 20–30 minutes |

## Potential cost drivers

Charges depend on API requests, Lambda invocations and duration, DynamoDB/SQS requests, Step Functions state transitions, SES messages, logs and CloudWatch resources. The storefront also uses hosting resources such as S3 and Lambda/API Gateway.

Do not assume that every service is free for the lab. Free Tier benefits and credits depend on the account and applicable program. Rounded or delayed Cost Explorer data does not establish a final cost of zero.

## Controlling usage

Start with a small test, avoid continuous submission loops, configure a budget and record the stacks you create. Clean up when evidence has been saved and the demonstration is no longer needed. Review retained resources and resources created outside the stacks separately.

The USD 5 workshop budget shown in section 5.2.5 is a configured budget amount, not the actual cost of this workshop. There is no need to generate more usage merely to make a cost appear in a screenshot.
<!-- workshop-detail:v1:end -->

## Choose a checkpoint between sessions

A good break follows a completed stack update and a result you can recover. After section 5.3, for example, record the stack, Region, queue names and test message ID. The next session can retrieve CloudFormation outputs and proceed to the worker. Leaving while a stack is rolling back makes the next session start with diagnosing that rollback.

Failure experiments spend time waiting for services. A queue message can be invisible, a workflow can be waiting for its next SES attempt, and CloudWatch needs a datapoint and an evaluation before an alarm changes state. Use that interval to record IDs and UTC times. Sending more messages during the wait changes the experiment.

## Estimate activity from service operations

An API call may enqueue a message, invoke Lambda and start several state transitions. A retry adds operations even when the order count stays constant. Record distinct orders, submissions, retries and log retention time alongside any billing estimate. These quantities help explain service-level charges later. Before leaving a session, identify which stacks the next session will need and which have finished their purpose.
