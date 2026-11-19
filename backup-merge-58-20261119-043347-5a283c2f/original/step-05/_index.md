---
title: "Revoke access"
weight: 5
chapter: false
pre: " <b> 5.8.5 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Do this step **last**, because once the key is deleted the CLI can no longer
be used:

1. Go to **IAM** → **Users** → select the user → **Security credentials** → **Deactivate** and then **Delete** the access key.
2. Detach the `OrderWorkshopDeployPolicy` policy from the user, then delete the policy if it is no longer needed.
3. Remove the profile from your machine: open the `%USERPROFILE%\.aws\credentials` and `config` files and delete the `order-workshop` section.
4. Remove the environment variables in PowerShell:

```powershell
Remove-Item Env:API_KEY, Env:CUSTOMER_EMAIL -ErrorAction SilentlyContinue
```
