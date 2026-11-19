---
title: "Install the tools"
weight: 2
chapter: false
pre: " <b> 5.2.2 </b> "
---

<!-- workshop-split:v1 -->

[← Section contents](../)

Open PowerShell and run:
```powershell
# Select Node.js 22 to match the template runtime.
winget install -e --id OpenJS.NodeJS.22
winget install -e --id Amazon.AWSCLI
winget install -e --id Amazon.SAM-CLI
```


Close and reopen PowerShell, then verify:

```powershell
node -v
aws --version
sam --version
```

<!-- workshop-detail:v1:begin -->
## What each tool does

| Tool | Purpose |
|---|---|
| AWS CLI v2 | Read configuration and operate AWS services from PowerShell |
| AWS SAM CLI | Build/deploy the application's SAM template |
| Node.js and npm | Work with JavaScript code and dependencies |
| Git | Track source and documentation changes |
| Hugo | Preview/build workshop documentation; it does not process AWS orders |

Reopen the terminal after installation to refresh PATH. Use a Node.js version compatible with the project and the Lambda runtime in its template. Installing a local Node.js version does not change a deployed Lambda runtime.

## Quick version checks

```powershell
aws --version
sam --version
node --version
npm.cmd --version
git --version
hugo version
```

Each tool should print its version; the CLI should be version 2. Hugo can be checked separately when only the AWS exercises are being performed. These commands do not create resources or send email.

## Common issues

For a command-not-found error, open a new terminal and check installation/PATH before changing AWS templates. Use PowerShell syntax on Windows rather than mixing it with Bash commands.

The screenshot below records four tool versions. It does not show npm or Hugo checks and does not establish AWS authentication or runtime compatibility.
<!-- workshop-detail:v1:end -->

<!-- workshop-proof:v1:begin -->
## Tool version evidence

{{< workshop-proof file="Version.png" alt="Tool version checks in PowerShell" caption="PowerShell shows AWS CLI 2.37.9, SAM CLI 1.166.2, Node.js v24.19.0 and Git 2.56.0.windows.1." >}}

The screenshot confirms that these four tools can be invoked on the local machine. It does not show npm or Hugo versions, establish AWS authentication or identify the deployed Lambda runtime.
<!-- workshop-proof:v1:end -->

## Find the executable that PowerShell selected

PowerShell resolves programs using PATH order. A machine with two Node.js installations or two AWS CLI versions can give different results in an old terminal and a fresh one. List the paths below and compare the Source column before reinstalling anything.

```powershell
Get-Command aws, sam, node, npm.cmd, git |
    Select-Object Name, Source, Version
```

The templates use `nodejs22.x`. If the SAM build reports an incompatible Node.js builder, select Node.js 22 on the build machine, start a new PowerShell window and recheck its version. On Windows, if the shell selects an npm PowerShell shim that cannot run, use the `npm.cmd` launcher supplied with Node.js. The file-writing and dependency examples later in the lab use that launcher.

## Read each check at its own stage

A version check shows that a local executable starts. The STS check on the next page identifies the AWS principal. Template validation in section 5.3 checks that SAM can interpret the project definition. Resolve a version lookup failure on the workstation first. When the executables report their versions, continue to the profile setup without creating a queue or submitting an email.
