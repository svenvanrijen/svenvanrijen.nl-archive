---
layout: post
title: Home Lab Automation (4) - PS> Install-Module -Name NewLabEnvironment -RequiredVersion 2.1
author: Sven van Rijen
---

During the last month, I was busy completing V2.0 (or actually v2.1 :wink:) of the NewLabEnvironment module. I promised to put it on steroids and in this blog, I'll tell you what this means.

# Home Lab Automation (4) - PS> Install-Module -Name NewLabEnvironment -RequiredVersion 2.1

-----

The biggest change compared to the old version, is that it’s possible now to hook your lab environment up directly to Azure Automation DSC. (Local DSC Pull servers will be supported in the near future...)

Why would you do that? Well, just creating ‘empty’ VM’s hasn’t got any purpose. If you’re building a lab environment to test a new product or to study for an exam, you want to have a fully functional environment. Let say a lab which includes Active Directory, DNS, a PKI environment, etc...

Building such an environment time after time is a very time consuming business. Luckily for us, Microsoft (and other vendors too) have created tools to take care of such tasks. And since we are PowerShell adepts (at least, I am), we are going to focus on PowerShell Desired State Configuration (DSC).

But first, what exactly has been changed to the model? The complete version history can also be found [here](https://github.com/ralpje/PowerShell-Lab-Module/blob/master/README.md).

### v2.1 (13-02-2017)

* New-LabVM
    * Fixed a bug regarding the default unattend.xml location on GitHub.

### v2.0 (12-02-2017)

Split up NewLabEnvironment into New-NATSwitch and New-LabVM.

* New-NATSwitch
    * New file, separated from New-LabVM.
    * No further changes.

* New-LabVM
    * Removed the NATName-parameter. Parameter is not being used within the script.
    * Changed URL-parameter to unattendloc-parameter. Since the unattend.xml file can be found both local and on the internet, unattendloc is a better name.
    * Changed to default location of the unattend.xml file to a directory within the main GitHub module.
    * Added the DSC-parameter.
    * Added the DSCPullConfig-parameter.
    * Added the NestedVirt-parameter.
    * Removed the SourceXML-parameter, since it was intended for interal use within the module only.
    * Added and updated Verbose statements. 

## DSCPullConfig

Let's focus on the DSCPullConfig-parameter first. 
There are a few prerequisites before heading off with the new version of the module. First, you’ll need an Azure subscription. Don’t know how to get started? Please visit [this website](https://azure.microsoft.com/en-us/get-started/).

When you have created your account, you can get started with Azure Automation DSC. On [this website](https://docs.microsoft.com/en-us/azure/automation/automation-dsc-getting-started) you’ll find an excellent walkthrough and some sample code to start with. 

What we would like to do is to [‘onboard’](https://docs.microsoft.com/en-us/azure/automation/automation-dsc-onboarding) our Lab VM’s for management by Azure Automation DSC.
Therefore, we need a base Azure Automation DSC meta configuration file, which will configure the local DSC config manager of the VM. Due to this configuration, the VM will know where to get its configuration. A sample of such a file, you can find on [Github](https://github.com/ralpje/PowerShell-Lab-Module/blob/master/Templates/AzureDSCPullConfig.ps1) and is displayed below:

```PowerShell
# The DSC configuration that will generate metaconfigurations
 [DscLocalConfigurationManager()]
 Configuration DscMetaConfigs
 {

     param
     (
         [Parameter(Mandatory=$True)]
         [String]$RegistrationUrl,

         [Parameter(Mandatory=$True)]
         [String]$RegistrationKey,

         [Parameter(Mandatory=$True)]
         [String[]]$ComputerName,

         [Int]$RefreshFrequencyMins = 30,

         [Int]$ConfigurationModeFrequencyMins = 15,

         [String]$ConfigurationMode = "ApplyAndMonitor",

         [String]$NodeConfigurationName,

         [Boolean]$RebootNodeIfNeeded= $False,

         [String]$ActionAfterReboot = "ContinueConfiguration",

         [Boolean]$AllowModuleOverwrite = $False,

         [Boolean]$ReportOnly
     )

     if(!$NodeConfigurationName -or $NodeConfigurationName -eq "")
     {
         $ConfigurationNames = $null
     }
     else
     {
         $ConfigurationNames = @($NodeConfigurationName)
     }

     if($ReportOnly)
     {
     $RefreshMode = "PUSH"
     }
     else
     {
     $RefreshMode = "PULL"
     }

     Node $ComputerName
     {

         Settings
         {
             RefreshFrequencyMins = $RefreshFrequencyMins
             RefreshMode = $RefreshMode
             ConfigurationMode = $ConfigurationMode
             AllowModuleOverwrite = $AllowModuleOverwrite
             RebootNodeIfNeeded = $RebootNodeIfNeeded
             ActionAfterReboot = $ActionAfterReboot
             ConfigurationModeFrequencyMins = $ConfigurationModeFrequencyMins
         }

         if(!$ReportOnly)
         {
         ConfigurationRepositoryWeb AzureAutomationDSC
             {
                 ServerUrl = $RegistrationUrl
                 RegistrationKey = $RegistrationKey
                 ConfigurationNames = $ConfigurationNames
             }

             ResourceRepositoryWeb AzureAutomationDSC
             {
             ServerUrl = $RegistrationUrl
             RegistrationKey = $RegistrationKey
             }
         }

         ReportServerWeb AzureAutomationDSC
         {
             ServerUrl = $RegistrationUrl
             RegistrationKey = $RegistrationKey
         }
     }
 }

 # Create the metaconfigurations
 # TODO: edit the below as needed for your use case
 $Params = @{
     RegistrationUrl = '<fill me in>';
     RegistrationKey = '<fill me in>';
     ComputerName = @('<some VM to onboard>', '<some other VM to onboard>');
     NodeConfigurationName = 'SimpleConfig.webserver';
     RefreshFrequencyMins = 30;
     ConfigurationModeFrequencyMins = 15;
     RebootNodeIfNeeded = $False;
     AllowModuleOverwrite = $False;
     ConfigurationMode = 'ApplyAndMonitor';
     ActionAfterReboot = 'ContinueConfiguration';
     ReportOnly = $False;  # Set to $True to have machines only report to AA DSC but not pull from it
 }

 # Use PowerShell splatting to pass parameters to the DSC configuration being invoked
 # For more info about splatting, run: Get-Help -Name about_Splatting
 DscMetaConfigs @Params
 ```
The New-LabVM function will rename and copy this file to the Windows\System32\Configuration directory of the LabVM. And because of the change of a registry key on the LabVM, the LabVM will register itself directly after the first boot.

```PowerShell
if ($DSC -eq $true)
    {
      # Copy DSC Pull Config to TEMP
      $destps1 = "$PSScriptRoot\temp\DSCPullConfig.ps1"
      Copy-Item -Path $DSCPullConfig -Destination $destps1
      Set-Location -Path "$PSScriptRoot\temp\"
        
      #Edit DSC Pull Config so it will contain the right node-name ($vmname)
      (Get-Content -Path '.\DSCPullConfig.ps1') -replace '\breplace\b', "$VMName" | Out-File -FilePath '.\DSCPullConfig.ps1'
        
      #Kick off DSC Pull Config to generate metamof
      Invoke-Expression -Command '.\DSCPullConfig.ps1'
             
      $sourcemof = "$PSScriptRoot\temp\DscMetaConfigs\$VMName.meta.mof"
      $destmof = "${VHD}:\Windows\system32\Configuration\MetaConfig.mof"
      Copy-Item -Path $sourcemof -Destination $destmof -Force

      Write-Verbose -Message "Copied metaconfig.mof to ${VHD}:\Windows\system32\Configuration\MetaConfig.mof"
      
      reg.exe load HKLM\Vhd ${VHD}:\Windows\System32\Config\Software
      Set-Location -Path HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies
      Set-ItemProperty -Path . -Name DSCAutomationHostEnabled -Value 2
      [gc]::Collect()
      reg.exe unload HKLM\Vhd
        
      Write-Verbose -Message "Enabled DSC Automation Host on $VMName."
    }
```

Key parameters you'll have to change to personalize this config are in the $Params block:

* `RegistrationUrl = '<fill me in>';`
* `RegistrationKey = '<fill me in>';`
* `ComputerName = @('<some VM to onboard>', '<some other VM to onboard>');`
* `NodeConfigurationName = 'SimpleConfig.replace';`

The values of these parameters can be found in your own Azure Automation DSC environment.
The RegistrationUrl and -Key are used to onboard the VM in a secure, certificate-based way. More info about where to find these values can be found [here](https://docs.microsoft.com/en-us/azure/automation/automation-dsc-onboarding). Look for _Secure Registration_.

In the example files for this module, ComputerName refers to the VMName parameter of the New-LabVM function. NodeConfigurationName refers to `<NameOfTheConfiguration>.$vmname.`.

When we take the example of the Web Server used on the Azure documentation website, the configuration file would look like this:

```PowerShell
configuration TestConfig
 {
     Node WebServer
     {
         WindowsFeature IIS
         {
             Ensure               = 'Present'
             Name                 = 'Web-Server'
             IncludeAllSubFeature = $true

         }
     }

     Node NotWebServer
     {
         WindowsFeature IIS
         {
             Ensure               = 'Absent'
             Name                 = 'Web-Server'

         }
     }
     }
```
(This TestConfig can also be found on [GitHub](https://github.com/ralpje/PowerShell-Lab-Module/blob/master/Templates/TestConfig.ps1).)

After compiling this configuration in Azure Automation DSC, you would refer to this config in your meta config as `NodeConfigurationName = 'TestConfig.replace';`. 

As you might have noticed, the New-LabVM function will change `.replace` in the config to the `$VMName` of the new VM.

To conclude this blog, I want to share with you the [script](https://github.com/ralpje/PowerShell-Lab-Module/blob/master/Templates/Compile_Azure_DSC_Node.config.ps1) I use when offline creating Azure Automation DSC configurations. In the Azure documentation I referred to earlier in this blog, a configuration is being uploaded and compiled by making use of the GUI. With this script, you can upload and compile directly from PowerShell (ISE). This is quite useful when making bulk changes to your configuration.

```PowerShell
#region Azure Login

Login-AzureRmAccount

#endregion

#region upload config



Import-AzureRmAutomationDscConfiguration -SourcePath "<fill me in>" `
                                         -Published `
                                         -ResourceGroupName "<fill me in>" `
                                         -AutomationAccountName "<fill me in>" `
                                         -Force

#endregion

#region compile config

$ConfigData = @{             
    AllNodes = @(             
        @{             
            Nodename = "*"             
        }

        @{             
            Nodename = "<fill me in>"
            Role = "<fill me in>"             
        }
        
       
   )             
}             

Start-AzureRmAutomationDscCompilationJob -ResourceGroupName "<fill me in>" `
                                         -AutomationAccountName "<fill me in>" `
                                         -ConfigurationName "<fill me in>" `
                                         -ConfigurationData $ConfigData

#endregion
```  