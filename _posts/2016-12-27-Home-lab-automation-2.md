---
layout: post
title: Home Lab Automation (2) - A general overview
author: Sven van Rijen
---

After a short holiday in New York City and Christmas, I’m ready to blog again! Before modifying the script any further, I first would like to give you a (graphical) overview of the environment we’re aiming to build with this script.

Despite the fact [Ralph already gave an overview of the technical environment the home lab script is designed for](http://www.365dude.nl/2016/05/06/building-a-home-lab-the-powershell-way/), I would like to give you an overview too. I think this will help you out when you're reading this blog and when you want to give the script a spin in your own little lab.

# Home Lab Automation (2) - A general overview

-----
So, what do you need to run this script?
![Graphical overview script](http://www.svenvanrijen.nl/images/Script_overview.jpeg)

First of all, hardware of course.  There are no real prerequisites regarding the hardware, although you must be able to run Hyper-V on it. Basically, this will come down to the system requirements of the Operating System you’re going to use. Also, the form factor of the device doesn’t matter at all. It’s all free to choose.
My own experience is that a rather small configuration (8GB of RAM and a 250GB HDD) is sufficient to test the basic functionality of the script. As in creating the NATSwitch and one or two VM’s.
If you really want to build a decent lab, then you probably want to make a (small) investment in some hardware. For a more decent lab experience, 16GB’s of RAM is a bare minimum. Besides that, 250GB of HDD can be sufficient. But if the number of VM’s grows, 500GB is a better option.

As mentioned earlier, you must be able to run Hyper-V on the machine, since this is the hypervisor this script is based on. Within Hyper-V you have a couple of possibilities to connect your machines to the internet or to an internal network. The NATSwitch we’re creating is actually best of both worlds.
The NATSwitch which is created by the script will be an internal switch which can translate (NAT) the internal traffic to the internet by making use of the internet connection of your host machine. Since the internal network and the network which your computer is residing in are on two separate subnets, the network will not interfere. This makes it possible to run a complete Windows server infrastructure including AD and DHCP without causing troubles on your own ‘Production’ network. If the NATSwitch is in place, you’re able to create VM’s which will be connected to this switch.

To save HDD space, the script uses a ‘main’ OS Disk. To create such a disk, you’ll have to create a VM running on the desired OS. Update this OS with the latest and greatest patches and run sysprep on it. Now you can remove the VM, but to not delete the created .vhdx HDD file. Mark this file as read-only as it will be used as the ‘main’ OS disk.
When creating any other VM in the lab, this ‘main’ OS disk will be used as a template. The .vhdx file belonging to the lab-vm will be an ‘differencing’ disk. With this setup, only the changes compared to the ‘main’ OS disk will be safed on the VM’s ‘differencing’ .vhdx file.

With the creation of the ‘main’ OS disk you have all the things in place to run the script. Now you can (re-)build your lab environment within minutes.


