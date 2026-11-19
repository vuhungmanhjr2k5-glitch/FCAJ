---
title: "Set up a cost alert"
weight: 5
chapter: false
pre: " <b> 5.2.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

1. Go to *Billing and Cost Management* → *Budgets* → *Create budget*.
2. Choose *Zero spend budget* or *Monthly cost budget* (for example, 5 USD).
3. Enter the email to receive alerts.

![Create budget](/images/5-Workshop/5.2-Prerequisite/004.png)

<!-- workshop-detail:v1:begin -->
## Purpose of a budget

AWS Budgets compares costs against a configured amount. Select a budget suitable for the lab; an example amount is not a guaranteed estimate of the workshop's final cost.

Budget notifications can be delayed and do not automatically stop all resources when a threshold is reached. Combine notifications with resource reviews and cleanup.

## Configuration to review

1. The budget type and reporting period.
2. The selected amount and notification thresholds.
3. The intended notification email address.
4. The scope: the whole account or a configured filter/tag selection.

Account-wide costs may include unrelated activity. Cost Explorer screenshots should show the date range and filters. Delayed or rounded figures do not establish that a lab is entirely free.

## After the exercise

Record stacks, buckets, log groups and retained resources. Clean up after evidence has been saved and the website is no longer needed for a demonstration. Do not generate extra emails or resources merely to populate a cost chart.

The supplied screenshot shows the workshop budget amount and its displayed status. It does not show actual spending, the reporting period, notification recipients or evidence that an alert email was delivered.
<!-- workshop-detail:v1:end -->

<!-- workshop-proof:v1:begin -->
## Budget configuration evidence

{{< workshop-proof file="Budget.png" alt="AWS Budgets list containing the workshop budget" caption="order-workshop-budget is configured at USD 5; the list shows Thresholds OK and Health status Healthy." >}}

USD 5 is the configured budget amount, not actual spending. My Monthly Cost Budget at USD 100 is a separate budget in the list. This screenshot does not show the reporting period, detailed notification thresholds, recipient email or delivery of an alert. Open order-workshop-budget to review those settings.
<!-- workshop-proof:v1:end -->

## Interpret a monthly threshold

For a worked example, a USD 5 monthly budget with an alert at 80% of actual cost has a USD 4 threshold within its scope. A **Forecasted** threshold evaluates the expected cost for the whole period. The same percentage can therefore produce notifications at different times. Record **Actual** or **Forecasted** alongside the amount.

If the account hosts other applications, inspect the budget filter first. An account-wide notification cannot identify which stack caused the increase. Select the matching dates in Cost Explorer, group by service and connect a change to an activity in the lab. Repeated delivery of malformed messages, for example, can increase worker invocations.

## Respond to a budget email

Record when you received the email and which billing period it covers. Check whether a test is still running, a stack is being kept for a demonstration, or a dashboard or log group is still needed. After cleanup allow for billing data to refresh before checking new charges. The period, threshold, scope and follow-up decision make a useful written record.
