---
layout: post
title: Automating WSUS installation through MDT
author: Sven van Rijen
---

Lately, I've been working with [Johan Arwidmark's](http://twitter.com/jarwidmark) [Hydration Kit For Windows Server 2016 and ConfigMgr Current / Technical Preview Branch](http://deploymentresearch.com/Research/Post/580/Hydration-Kit-For-Windows-Server-2016-and-ConfigMgr-Current-Technical-Preview-Branch). A great tool to enroll a full blown SCCM and/or MDT deployment.

# Automating WSUS installation through MDT

One of the servers you can deploy with this tool is a WSUS server. What struck me was that despite the task sequence to perform the base installation of the server is available within MDT, WSUS itself isn't installed or configured.


## Prerequisites

Installing WSUS isn't a straight forward process. Although it looks like 'Windows Server Update Services' is just a two check mark operation, reality is a bit different. A lot of prerequisites have to be met too, to be able to install this feature:

- [ ] Roles
    - [X] Web Server (IIS)
        - [X] Web Server
            - [X] Common HTTP Features
                - [X] Default Document
                - [X] Static Content
            - [X] Performance
                - [X] Dynamic Content Compression
            - [X] Security
                - [X] Request Filtering
                - [X] Windows Authentication
            - [ ] Application Development
                - [X] .NET Extensibility 4.6
                - [X] ASP .NET 4.6
                - [X] ISAPI Extensions
                - [X] ISAPI Filters
            - [ ] FTP Server
                - [X] FTP Service
            - [X] Management Tools
                - [X] IIS Management Console
                - [X] IIS 6 Management Compatibility 
                    - [X] IIS 6 Metabase Compatibility
    - [ ] Windows Server Update Services
        - [X] WSUS Services
        - [X] SQL Server Connectivity
- [ ] Feature
    - [X] .NET Framework 4.6 Features
        - [X] ASP.NET 4.6
    - [X] WCF Services
        - [X] HTTP Activation
    - [X] Remote Server Administration Tools
        - [X] Role Administration Tools
            - [X] Windows Server Update Services Tools
                - [X] API and PowerShell cmdlets
                - [X] User Interface Management Console
    - [X] Windows Process Activation Service
        - [X] Configuration APIs

By default, WSUS relies on the Windows Internal Database (WID) feature. But in this scenario, I've chosen to make use of the already available SQL Server Express installation on WSUS01.

## Post-installation tasks

Just adding this set of roles and features to WSUS01 isn't enough to automate the entire installation. We still have to run manually through the post-installation tasks. To avoid that we're going to create a PowerShell 'script', which we can later add as an application to the tasks sequence.

What this script will do, is run the wsusutil.exe postinstall tasks. The SQL Instance Name will be configured to use the local SQLExpress database instance `WSUS01\SQLExpress`. The Content Directory (the location where WSUS will store the actual updates) will be configured as `E:\WSUS\`.  

```Powershell
<#
Solution: Hydration
Purpose: Used to postinstall configure Windows Update Services 
Version: 1.0 - May 22, 2017

This script is provided "AS IS" with no warranties, confers no rights and 
is not supported by the authors or Deployment Artist. 

Author - Sven van Rijen
    Twitter: @svenvanrijen
    Blog   : .
#>

# Determine where to do the logging 
$tsenv = New-Object -COMObject Microsoft.SMS.TSEnvironment 
$logPath = $tsenv.Value("LogPath") 
$logFile = "$logPath\$($myInvocation.MyCommand).log" 

# Start the logging 
Start-Transcript $logFile 
Write-Host "Logging to $logFile" 

# Start the postinstall config of WSUS
Set-Location 'C:\Program Files\Update Services\Tools'
.\WsusUtil.exe postinstall SQL_INSTANCE_NAME=WSUS01\SQLExpress Content_Dir=E:\WSUS\

# Stop logging 
Stop-Transcript
```

Save this script as `c:\setup\Configure - WSUS\Configure-WSUS.ps1`.

The script can also be found [here](https://github.com/svenvanrijen/HydrationKit/tree/master/HydrationCMWS2016v2/Configure%20-%20WSUS).

After creating this script, add a new application to the Applications section in the Deployment Workbench tool.

![Add application](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/workbench_applications.png)

- Right-click `Applications` and choose `New Application`. Use the following settings in the `New Application Wizard`:
    - Application with source files 
    - Publisher: `<blank>` 
    - Application name: Configure - WSUS 
    - Version: `<blank>` 
    - Source Directory: C:\Setup\Configure - WSUS 
    - Specify the name of the directory that should be created: Configure - WSUS
    - Command Line: powershell.exe -Command "set-ExecutionPolicy Unrestricted -Force; cpi '%DEPLOYROOT%\Applications\Configure - WSUS\Configure-WSUS.ps1' -destination c:\; c:\Configure-WSUS.ps1; ri c:\*.ps1 -Force; set-ExecutionPolicy Restricted -Force"
    - Working directory: `<default>`  


## Customize the task sequence

Based on the info above, we're able to customize the default WSUS01 task sequence of the Hydration Kit in the Deployment Workbench tool.

- Select the 'Task Sequences' section in the Deployment Workbench tool.

- Right-click the 'WSUS01 - Full Installation' task sequence, and click `Properties`.

![WSUS01 - Full Installation task sequence](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/workbench_ts_wsus01.png)

- First, create a new group after the 'Install - SQL Server Management Studio' item. Name this group 'Install - WSUS'.

![Add WSUS installation steps](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/ts_wsus.png)

- Within this group, add a new `Roles` > `Install Roles and Features` task. Name this task 'Install - WSUS'.

- Select the features and roles as described [above](#prerequisites).

- Click `Apply`.

- After the newly created task, create another task (`Add` > `General` > `Install Application`) and name this task `Configure - WSUS`. Use the following settings:
    - Name: Configure - WSUS
    - Install a Single Application: Configure - WSUS

- After the Install - Microsoft Visual C++ 2015 - x86-x64 action, add a Computer Restart action.

- Click `OK`.

Now you are ready to deploy WSUS01:
- Hard drive: 300 GB (dynamic disk)
- Memory: 2 GB RAM minimum, 4 GB recommended