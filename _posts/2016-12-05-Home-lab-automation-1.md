---
layout: post
title: Home Lab Automation (1) - Introduction
author: Sven van Rijen
---

On the 22nd of November, I visited [Experts Live](http://www.expertslive.nl) in Ede, The Netherlands. There were lots of good sessions, but one of them got my attention in particular. And that was the one on automating the (re-)building of your home lab, presented by [Ralph Eckard](http://www.365dude.nl). Fortunately, Ralph shared his scripts on [GitHub](https://github.com/ralpje/Experts-Live-2016), so I could start work with it myself. 

# Home Lab Automation (1) - Introduction 

-----

First, I have to compliment Ralph on his work, he laid a strong foundation for a very handy script. Although the script is written with a lab environment in mind, it can be used in a Production scenarios as well. 
If you would like to learn more about the original script, please read [Ralph's blogpost](http://www.365dude.nl/2016/05/06/building-a-home-lab-the-powershell-way/) about it. He also mentions the (quite cheap) hardware he is using for his lab setup.


When I gave the script the first spin myself, I noticed that the IP address of the Gateway was hardcoded in the script. That's fine when you are using the exact same address space as the original script. However, for me that was not the case and I wanted to be able to change the IP-address, DNS server and Gateway for the new lab VM in one go.
So I changed some lines in the original `Create-LabServer.ps1` script.

First, I added a Gateway variable to the script:

```powershell
#region define variables
# User defined variables that varies per server
$VMName = 'EL-DEMO-F01'
$VMIP = '172.16.0.201/24'
$DNSIP = '172.16.0.200'
$GWIP = '172.16.0.1'
```

Just as the other variables, also the Gateway variable has to be 'injected' into the `unattended.xml' file:

```powershell
# add XML Content
[xml]$Unattend = Get-Content $SourceXML
$Unattend.unattend.settings[1].component[2].ComputerName = $VMName
$Unattend.unattend.settings[1].component[3].Interfaces.Interface.UnicastIpAddresses.IpAddress.'#text' = $VMIP
$Unattend.unattend.settings[1].component[3].Interfaces.Interface.Routes.Route.NextHopAddress = $GWIP
$Unattend.unattend.settings[1].component[4].Interfaces.Interface.DNSServerSearchOrder.IpAddress.'#Text' = $DNSIP
```

With these two lines added to the file, we're now able to deploy a VM with the correct Gateway address. But, Ralph did also create a function (`new-labvm function.ps1`), so also in this files the mentioned variables are added.
This makes it possible, after loading the function into memory, to create a new lab VM with a single PowerShell command:

`New-LabVM -VMName Hostname -VMIP 192.168.123.20/24 -DNSIP 8.8.8.8 -GWIP 192.168.123.1`.

Ralph merged my Pull request into the master repo, so all changes mentioned in this post are directly available on [GitHub](https://github.com/ralpje/Experts-Live-2016).

