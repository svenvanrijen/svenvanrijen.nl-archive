---
layout: post
title: Home Lab Automation (5) - A full run of the module, step by step
author: Sven van Rijen
---

Last Thursday, 9th March 2017, I was presenting at the 10th DuPSUG meeting. Unfortunately, because I had to much manual actions in my demo, I couldn't finish it. That's way I will describe a full run of the New-LabEnvironment module in this blog post. This will also be the last blog post regarding _functionality_ of the module. On which subjects I will continue to blog? You'll find out at the end of this post.

# Home Lab Automation (5) - A full run of the module, step by step

-----
(The slide deck of my presentation and all other source files can be found on [GitHub](https://github.com/DuPSUG/DuPSUG10/tree/master/RalphEckhardSvenvanRijen).)

Let's begin at the beginning:

### Prerequisite

When you want to utilize the Azure Automation DSC part of the module, you'll need a Azure subscription. You can [try Azure for free](https://azure.microsoft.com/en-us/free/) and receive $200 credit when you sign up [here](https://azure.microsoft.com/en-us/free/).

### Step 1

We are going to start with installing both the new (Resource Manager) and old (Service Management/Classic) Azure PowerShell modules. We're going to need both further down the line.

```Powershell
Install-Module -Name AzureRM

Install-Module -Name Azure
```

### Step 2

With these modules installed, we can login to the AzureRM environment through PowerShell

```Powershell
Login-AzureRmAccount
```

Please supply your username and password in the popup browser page.

### Step 3

Logged in, we can supply a couple of variables used throughout the script.

```Powershell
$resourcegroupname = "DuPSUG10"

$location = "West Europe"

$AutomAccountName = "DuPSUG10"
```

With these variables, we can create both AzureRM Resource Group and Azure RM Automation Account:

```Powershell
New-AzureRmResourceGroup -Name $resourcegroupname -Location $location

New-AzureRmAutomationAccount -Name $AutomAccountName `
                             -ResourceGroupName $resourcegroupname `
                             -Location $location
```

### Step 4

From this new Azure RM Automation account, we are going to need the Registration Info to supply in our DSC meta config in 'Step 5'. To get the details we can run

```Powershell
Get-AzureRmAutomationRegistrationInfo -ResourceGroupName $resourcegroupname `
                                      -AutomationAccountName $AutomAccountName
```

Now we have the Registration URL and Registration Key we need in our Azure DSC meta config.

### Step 5

Open your Azure DSC meta config (a template can be found [here](https://github.com/ralpje/PowerShell-Lab-Module/tree/master/Templates)). At the end of the file, you'll find the Params-block:

```Powershell
# Create the metaconfigurations
 # TODO: edit the below as needed for your use case
 $Params = @{
     RegistrationUrl = '<fill me in>';
     RegistrationKey = '<fill me in>';
     ComputerName = 'replace';
     NodeConfigurationName = '<nodeconfig name here>.replace';
     RefreshFrequencyMins = 30;
     ConfigurationModeFrequencyMins = 15;
     RebootNodeIfNeeded = $False;
     AllowModuleOverwrite = $False;
     ConfigurationMode = 'ApplyAndMonitor';
     ActionAfterReboot = 'ContinueConfiguration';
     ReportOnly = $False;  # Set to $True to have machines only report to AA DSC but not pull from it
 }
```

Within this block, replace the `<fill me in>` of `RegistrationUrl` and `RegistrationKey` with the Registration Info you have discovered in `Step 4`.

### Step 6

Time for the 'real work': the actual DSC config. Again, there is a [template available](https://github.com/ralpje/PowerShell-Lab-Module/tree/master/Templates), as well as a version I had prepared for the [DuPSUG demo](https://github.com/DuPSUG/DuPSUG10/tree/master/RalphEckhardSvenvanRijen).
Feel free to create your own version of the config. Please remember to use the name of your config in both the parameter block mentioned in `Step 5` and when uploading and compiling the config in `Step 11`. 

For the demo, I've used a config which creates an AD DC, including a DHCP and DNS server. For this config, we need to install several DSC resources both on our local computer as within the Azure Automation account online (`step 7`).

```Powershell
Install-Module -Name xActiveDirectory, `
                     xNetworking, `
                     xDnsServer, `
                     xPendingReboot, `
                     xDHCPServer, `
                     xPSDesiredStateConfiguration
```

### Step 7

When creating a new domain and/or forest, you'll have to supply credentials to be used during that process. We don't want to have these credentials in plain text in our config. Not even in a lab environment. That's why we will add these credentials to the Azure Automation Account to be stored safely. When compiling the config, the credentials will be picked up out the Automation Account and stored encrypted within the MOF file.

To store the credentials, we first have to login to the `Classic` Azure context. 
```Powershell
Add-AzureAccount
```

Within this context, add a Automation account.

```Powershell
New-AzureAutomationAccount -Name $AutomAccountName -Location $location
```

Now we can generate a Local Domain Admin username and password.

```Powershell
$user = "dupsug.com\Administrator"

$pw = ConvertTo-SecureString "P@ssW0rd" -AsPlainText -Force

$cred = New-Object –TypeName System.Management.Automation.PSCredential `
                   –ArgumentList $user, $pw

New-AzureAutomationCredential -AutomationAccountName $AutomAccountName `
                              -Name "Local domain admin" `
                              -Value $cred
```

### Step 8

As you might have noticed, up till now we didn't logged in to the [Azure Portal](https://portal.azure.com). All tasks could (luckily) be performed through PowerShell. For the next task, we have to login to the portal. Although, there is a alternative way as [Ben Gelens has described in a blog](https://channel9.msdn.com/Blogs/MVP-Azure/Azure-Automation-DSC-Part-6-Advanced-Configurations). But for now, we stuck with the manual way.

When logged in to the portal, go to the automation account and click `Modules Gallery`.

![Modules Gallery](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/blog5-step8-01.jpg "Modules Gallery")

Within the `Modules Gallery`, search for a module to install. In this example `xActiveDirectory`. Select the module, 

![Select module to install](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/blog5-step8-02.jpg "Select module to install")

click `Import` 

![Click Import](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/blog5-step8-03.jpg "Click Import")

and `OK`.

![Click OK](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/blog5-step8-04.jpg "Click OK")

Repeat these steps for each of the (in this demo) six modules.

### Step 9

Next step is to upload the DSC config to Azure Automation DSC. First, login again within the `AzureRM` context.
```Powershell
Login-AzureRmAccount
```

And upload the config.
```Powershell
Import-AzureRmAutomationDscConfiguration -SourcePath "C:\Users\SvenvanRijen\OneDrive\DuPSUG 10\DuPSUG_domain.ps1" `
                                         -Published `
                                         -ResourceGroupName $resourcegroupname `
                                         -AutomationAccountName $AutomAccountName `
                                         -Force
```

### Step 10

Now the config have to be compiled. In other words, a MOF file has to be created from the config.

Please wait for all modules added in `Step 8` to be `Available`.

![Available](https://svenvanrijen.github.io/svenvanrijen.nl-archive/images/blog5-step10-01.jpg "Available")


```Powershell
$ConfigData = @{             
    AllNodes = @(             
        @{             
            Nodename = "*"             
            DomainName = "dupsug.com"             
            RetryCount = 20              
            RetryIntervalSec = 30
            ConfigurationMode = 'ApplyAndAutoCorrect'            
            PsDscAllowPlainTextPassword = $true
          }

        @{             
        Nodename = "DC01"             
        Role = "Primary DC"             
        DomainName = "dupsug.com"             
        RetryCount = 20              
        RetryIntervalSec = 30            
        PsDscAllowPlainTextPassword = $true
        PsDscAllowDomainUser = $true        
       }
     )
}

Start-AzureRmAutomationDscCompilationJob -ResourceGroupName $resourcegroupname `
                                         -AutomationAccountName $AutomAccountName `
                                         -ConfigurationName "DuPSUG_domain" `
                                         -ConfigurationData $ConfigData
```

This will return some feedback, including a `JobID`.

### Step 11

With the `JobID` from `Step 10`, we can get the job status.

```Powershell
Get-AzureRmAutomationJob -Id <fill me in> `
                         -ResourceGroupName $resourcegroupname `
                         -AutomationAccountName $AutomAccountName
```

As soon this job is `Succeeded` we can continue with `Step 12`.

### Step 12

If not installed already, install the `New-LabEnvironment` module on your local system.

```Powershell
Install-Module NewLabEnvironment
```

With this module installed, we can generate both the NATSwitch... 

```Powershell
New-NATSwitch -Name dupsug `
              -IPAddress 10.0.1.1 `
              -PrefixLength 24 `
              -NATName dupsug `
              -Verbose
```

and LabVM.

```Powershell
New-LabVM -VMName DC01 `
          -VMIP 10.0.1.100 `
          -GWIP 10.0.1.1 `
          -diskpath "C:\vhdx\" `
          -ParentDisk "C:\vhdx\W2K16_Template.vhdx" `
          -VMSwitch dupsug `
          -DNSIP 8.8.8.8 `
          -DSC $true `
          -DSCPullConfig "<point to>\DuPSUGDSCPullConfig.ps1" `
          -NestedVirt $false `
          -Verbose
```

### Finished! :smiley:

After the first boot into the OS, the new LabVM will automatically register itself into Azure Automation DSC and pull down its MOF file. And the magic begins...

## Up next
As stated at the beginning of this blog post, this will (probably) be the last blog post regarding _functionality_ of the NewLabEnvironment module. But, there are still things to do. There are at least two more things on my 'wishlist' for this module:

1. Break up the module in smaller pieces and add Pester tests to it.
2. Add 'Continuous Integration' to it in some form ([AppVeyor](https://www.appveyor.com/), [Teamcity (via Powershell.org)](https://powershell.org/build-server/)).

I guess this will still keep me busy for a while :wink:.