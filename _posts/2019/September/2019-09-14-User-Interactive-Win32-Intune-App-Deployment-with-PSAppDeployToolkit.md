---
layout: post
date: 2019-09-14
title:  "User Interactive Win32 Intune App Deployment with PSAppDeployToolkit"
tags: [Intune, Application Packaging ,Azure ,Windows 10 ,EMS ,PSAppDeployToolkit]
banner: "/SemiAnnualChat/assets/UserDrivenWin32/win32banner.png"
image: "assets/UserDrivenWin32/Win32Twitter.png"
description: "Use Intune Win32 Application deployment and the PowerShell App Deployment Toolkit to provide a user friendly application deployment for required applications"
author: "Stefan van der Busse"
comments: true
youtubeId: x3-svH1b6WE
---
___

Win32 application deployments using Microsoft Intune continues to improve, with application dependencies now available and script based requirement capabilities, Intune provides a comprehensive cloud based application deployment solution hot on the heels of it's big brother Configuration Manager.

One feature that does not have parity with SCCM, is the option to [allow users to interact](https://docs.microsoft.com/en-us/sccm/core/plan-design/changes/whats-new-in-version-1802#allow-user-interaction-when-installing-an-application) with a installation prompts during deployment.
There has been some great community developed tools like the [PSAppDeployToolkit](https://psappdeploytoolkit.com/) which can provide end users with company branded dialogue boxes, creating a more personable deployment experience.

![PSAppDeployToolkit Example](/SemiAnnualChat/assets/UserDrivenWin32/Win32PSApp2.png "PSAppDeployToolkit Example")

Of course, we can use the PSAppDeployToolkit with Intune on it's own today, utilising it's superior logging and PowerShell based installation cmdlets to silently install .msi or .exe applications. But as Win32 applications are installed from within the system (session 0) context, we are unable to benefit from the user driven dialogue boxes.

It is common for applications to have dependencies that need to be closed during installation. And as Administrators we need to make sure that the application installs successfully, and within a timely manner. But also need to make sure that to achieve these objectives, that we don't do so at the users expense.

As Administrators we can be force close dependent applications prior to installation, using some crude stop-process commands coupled with a force parameter. But without the ability to set maintenance Windows within Intune, or being able to specify installation to take place outside of Active Hours, we're going to get some very unhappy end users as applications close without the opportunity to defer installation to a time that suits them better.

## The Problem to Solve?
> How can we provide end users whose devices are managed with Microsoft Intune (Standalone) with a better installation experience, for high user applications while not relying on them to self serve application updates using the Company Portal?

## The Solution
Using a combination of PowerShell, PSAppDeployToolkit and a tool borrowed from MDT called ServiceUI - we can provide a user friendly deployment experience from a Win32 Intune required application assignment, like the one I demonstrate in this video:

___
{% include youtubePlayer.html id=page.youtubeId %}
___

## Prerequisites:
* Microsoft 365 (any SKU), EMS E3 or Intune Device Licensing
* The PSAppDeployToolkit downloaded from [Github](https://github.com/PSAppDeployToolkit/PSAppDeployToolkit)
* ServiceUI.exe, borrowed from Microsoft Deployment Toolkit (MDT)
* Microsoft Win32 Content Prep Tool from [Github](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool)

For those unfamiliar with some of components listed in the prerequisites, here's a brief overview:

**ServiceUI.exe**

![ServiceUI.exe in DeploymentShare](/SemiAnnualChat/assets/UserDrivenWin32/Win32PSApp10.png "ServiceUI.exe in DeploymentShare")

ServiceUI.exe is a tool included as part of MDT, it allows us to to launch applications from within the system context, into the context of an active console user. This allows end users to interact with individual applications running as system, rather than being elevated themselves. The tool can be found in the Tools\x64 directory of a deployment share.

**PSAppDeployToolkit**

![PSAppDeployToolkit Example](/SemiAnnualChat/assets/UserDrivenWin32/Win32PSApp2.png "PSAppDeployToolkit Example")

The [PSAppDeploymentToolkit](https://psappdeploytoolkit.com/) website describe their toolkit as:

> "The PowerShell App Deployment Toolkit provides a set of functions to perform common application deployment tasks and to interact with the user during a deployment. It simplifies the complex scripting challenges of deploying applications in the enterprise, provides a consistent deployment experience and improves installation success rates.
The toolkit functions as a wrapper"

Definitely check out the [website](https://psappdeploytoolkit.com/) to find a full list of features and capabilities.

**Microsoft Win32 Content Prep Tool**

The content prep tool allows Intune Administrators to wrap install files for Win32 Applications, and use silent install switches or custom install scripts to install the wrapped application.

## Solution Breakdown
1. Intune runs the the Configure.ps1 PowerShell Script (shown below)
2. PowerShell uses WMI to check to see if any users are running the target process
3. If the process is not detected, PSAppDeployToolkit is run in **noninteractive** mode, and the installation/upgrade takes place unknowingly to the user
4. If the process is detected, PSAppDeployToolkit is launched in **interactive mode** within the users console session based on the user running explorer.exe using ServiceUI, prompting the user to close the required applications before starting the installation
5. If the user chooses to defer the installation, exit code 60012 is returned to Intune - and a failure is recorded.

**Retry vs Failure**

Why return a failure to Intune rather a retry? The retry time on a Win32 application is only 5 minutes, which is not a long enough delay before prompting the user who just dismissed the prompt to try the installation again. If a failure is returned, Intune will wait at least 24 hours before retrying in the installation again.

**Configure.ps1**

Here is an example of a PowerShell script that can be used to detect the target process is in use, and launch PSADT in either interactive, or non-interactive mode accordingly:

```powershell
$targetprocesses = @(Get-WmiObject -Query "Select * FROM Win32_Process WHERE Name='7zFM.exe'" -ErrorAction SilentlyContinue)
if ($targetprocesses.Count -eq 0) {
    Try {
        Write-Output "No user logged in, running without SerivuceUI"
        Start-Process Deploy-Application.exe -Wait -ArgumentList '-DeployMode "NonInteractive"'
    }
    Catch {
        $ErrorMessage = $_.Exception.Message
        $ErrorMessage
    }
}
else {
    Foreach ($targetprocess in $targetprocesses) {
        $Username = $targetprocesses.GetOwner().User
        Write-output "$Username logged in, running with SerivuceUI"
    }
    Try {
        .\ServiceUI.exe -Process:explorer.exe Deploy-Application.exe
    }
    Catch {
        $ErrorMessage = $_.Exception.Message
        $ErrorMessage
    }
}
Write-Output "Install Exit Code = $LASTEXITCODE"
Exit $LASTEXITCODE
```
**Wrapping It All Together**

Here's the directory to wrap using the Win32 Content Prep Tool with the three components:
* Configure.ps1 PowerShell Script
* PSAppDeployToolkit, customised for my application
* ServiceUI, borrowed from MDT

![Win32 Directory to Wrap](/SemiAnnualChat/assets/UserDrivenWin32/Win32PSApp1.png "Win32 Directory to Wrap")

**Intune Install Command:**

The install command used in Intune, is (a reminder that PowerShell scripts launched via Win32 are started in x86, rather x64).
```
powershell.exe -executionpolicy bypass -file ".\Configure"
```
For extra points, you can add some switching logic in your configure script and pass a parameter that tells the script whether to launch PSADT in Install or Un-Install mode.
```
powershell.exe -executionpolicy bypass -file ".\Configure" -Mode Install
```

![Intune install command](/SemiAnnualChat/assets/UserDrivenWin32/Win32PSApp6.png "Intune install command")

## Summary

This solution is not a new one, previously used by SCCM administrators prior to 1802, but I've yet to see it used with Intune. It has a couple of moving parts, but is relatively simple once you wrap your head around the components.

Recently, I utilised this deployment method with 3 separate customers, upgrading a critical database client application used by 75% of all end users, and within a 24 hour period - we were on track for 80% upgrade success. And most importantly, across 600 client machines, we received about 3 support calls.

The previous upgrade method for these customers would have been either a manual per user upgrade, or a deploying the upgrade via Group Policy, which depended on the user being on-site at start up time.

![Win32 Directory to Wrap](/SemiAnnualChat/assets/UserDrivenWin32/Win32PSApp3.png "Win32 Directory to Wrap")

Hopefully this provides some inspiration into what is possible with Win32 App Deployment through Intune. If you'd like more details, ask me a question in the comments or send me a tweet on twitter.