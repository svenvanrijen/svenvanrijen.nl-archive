---
layout: post
title: Home Lab Automation (3) - PS> Install-Module -Name NewLabEnvironment
author: Sven van Rijen
---

Hi there! On the last day of 2016, I've published a module version of the home lab script on the Powershell Gallery. So, this means easy download and installation, also when a new version is released. Which happened today, since I've just uploaded v1.1 of the module to the Gallery. In this blogpost, I'll walk you through the installation and execution of the module.

# Home Lab Automation (3) - PS> Install-Module -Name NewLabEnvironment

-----

So, first things first: the installation itself. For those who already installed modules from the Gallery, this will sound familiar. `Install-Module -Name NewLabEnvironment` does the trick. You might get a pop-up when installing a module from the Gallery (or another source) for the first time. Check if you really trust the source of the code you are going to download, and then just click 'Yes' or 'Yes to all' to continue the installation.

After installation, you can check if the module is indeed correctly installed by running `Get-Module "NewLabEnvironment"`. This will return something like this:

``` Powershell
ModuleType Version    Name                                ExportedCommands                                                                    
---------- -------    ----                                ----------------                                                                    
Script     1.1        NewLabEnvironment                   {New-LabVM, New-NATSwitch}    
```

As you can see, both parts of the original script (New-NATSwitch and New-LabVM) are part of the module. Now, how can you execute this module?

## New-NATSwitch
I'll first give an example of the `New-NATSwitch` part of the module. As you might remember ([part 1](http://www.svenvanrijen.nl/Home-lab-automation-1/) and [part2](http://www.svenvanrijen.nl/Home-lab-automation-2/) of this blog), creating the new NATSwitch involved several lines of code. Also, certain results of one line had to be captured to be used in another line of code. I've all combined this in the module, which makes it possible to create a NATSwitch by running just one line of code:

``` Powershell
New-NATSwitch -Name test `
              -IPAddress 10.0.1.1 `
              -PrefixLength 24 `
              -NATName testnat `
              -Verbose
```

In the background, this code is being executed:
``` Powershell
New-VMSwitch -SwitchName $Name -SwitchType Internal

    $NetAdapter = Get-NetAdapter -Name "vEthernet ($Name)"
    $InterfaceIndex = $NetAdapter.ifIndex
          
    New-NetIPAddress -IPAddress $IPAddress -PrefixLength $PrefixLength -InterfaceIndex $InterfaceIndex
          
    $ip2 = $IPAddress.Split('.')
    $ip2[-1] = "0/$PrefixLength"
    $InternalIPInterfaceAddressPrefix = $ip2 -join '.'

    New-NetNat -Name $NATName -InternalIPInterfaceAddressPrefix $InternalIPInterfaceAddressPrefix 
```

In this case, a NATswitch with the name `test` is being created. The switch will have IPAddress `10.0.1.1` and a PrefixLength of `24` (255.255.255.0). Also, the correct NAT network is created (`testnat`). As you've might have noticed: you no longer have to look for `$InterfaceIndex` or the `$InternalIPInterfaceAddressPrefix` parameters yourself. Those parameters are automatically created by the module.

## New-LabVM
Now you have the NATSwitch available, you can start creating LabVM's:

``` Powershell
New-LabVM -VMName DC01 `
          -VMIP 10.0.1.100 `
          -GWIP 10.0.1.1 `
          -diskpath "C:\vhdx\" `
          -ParentDisk "C:\vhdx\W2K12R2_Template.vhdx" `
          -VMSwitch test `
          -DNSIP 8.8.8.8 `
          -NATName testnat `
          -Verbose
```

With this example, a VM with the name `DC01` will be created. The VM will have IPAddress `10.0.1.100`, will be based on a Windows Server 2012R2 installation and will be connected to the earlier created NATSwitch `test`. This is all still quite familiar with the original script. 
But, to set the name and IPAddress of the new VM, the `unattend.xml` file is being used. The source of the fill which will be minipulated with the values you enter, is by default located on GitHub and downloaded when the script is being executed. For lab purposes, the content of this file will be sufficient. Although, I can imagine that there are cases that you a more personalized version of the `unattend.xml` file. Now, in v1.1 of the module, the URL (or local file location) of the `unattend.xml` is become an optional parameter. In the case, you want to link to a personalized version of `unattend.xml`, you can just add a parameter to your script:

``` Powershell
New-LabVM -VMName DC01 `
          -VMIP 10.0.1.100 `
          -GWIP 10.0.1.1 `
          -diskpath "C:\vhdx\" `
          -ParentDisk "C:\vhdx\W2K12R2_Template.vhdx" `
          -VMSwitch test `
          -DNSIP 8.8.8.8 `
          -NATName testnat `
          -URL "c:\files\unattend.xml"
          -Verbose
```
Now the local version of the `unattend.xml` file in `C:\files\` will be used.


## Teaser :wink:
In the next blogpost, I'll put the whole script 'on steroids' :muscle:...