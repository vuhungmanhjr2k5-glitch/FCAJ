---
title: "Create the project folder"
weight: 6
chapter: false
pre: " <b> 5.2.6 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

```powershell
$appDir = Join-Path $env:USERPROFILE 'Downloads\order-system'
New-Item -ItemType Directory -Path $appDir -Force -ErrorAction Stop | Out-Null
Set-Location -LiteralPath $appDir -ErrorAction Stop
New-Item -ItemType Directory -Path .\src -Force -ErrorAction Stop | Out-Null
```
In the following sections, we will add `template.yaml` and the Lambda source code to this folder.

<!-- workshop-detail:v1:begin -->
## Organize application code and documentation

Keep the AWS workshop project, Moc storefront package and Hugo documentation distinct. Templates and Lambda code belong to the AWS project; _index.md/_index.vi.md pages and evidence images belong to Hugo. Editing documentation does not require an AWS redeployment.

| Path under `order-system` | Contents |
|---|---|
| `template.yaml` | SAM application resources |
| `src/index.mjs` | Order-processing Lambda handler |
| `src/package.json` | Worker dependencies |
| `samconfig.toml` | Settings saved after guided deployment |

This is the intended structure. Files are created as the workshop progresses; they do not all need to exist immediately after creating the project directory.

## Check the current directory

```powershell
Get-Location
Get-ChildItem
```

Before building or deploying, confirm that the working directory contains the intended template. Reuse and inspect existing files from previous exercises instead of replacing working code with empty files.

<!-- workshop-detail:v1:end -->

## Return to the correct project in a later session

The walkthrough uses two sibling directories under Downloads. `Workshop-AWS-ZIP` contains the Hugo site you are reading. `order-system` holds the AWS template, worker and measurement tools you will create. From a new PowerShell window, the following block enters the application directory and lists its top-level files.

```powershell
$appDir = Join-Path $env:USERPROFILE 'Downloads\order-system'
Set-Location -LiteralPath $appDir -ErrorAction Stop
Get-Location
Get-ChildItem -Name
```

An empty `src` directory is expected at the end of section 5.2. After 5.3.1, look for `template.yaml` next to `src`. After 5.4.2, look for `src/index.mjs`. Saving settings during a guided deployment adds `samconfig.toml` beside the template. A directory containing `content`, `themes` or `static` is the documentation project.

## Resume a partially completed application

Read the beginning of an existing file before pasting a block that replaces it. For example, a template with a `SenderEmail` parameter has already reached the orchestration stage. Save local changes in Git or make a copy before using a full-file example. For a build issue, record the working path and the filename you changed. Those two details usually narrow the search more quickly than a screenshot of the editor.
