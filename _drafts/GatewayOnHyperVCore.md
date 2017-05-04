---
layout: single
title: Creating a Windows Server 2016 Core Gateway Router
#excerpt: "Because"
tags: [WindowsServer2016, ServerCore, Powershell]
date: 2017-04-28 00:00:00
# last_modified_at: 2017-04-29 00:00:00
comments: false
share: true
header:
  image: /assets/images/home-banner.jpg
  caption: "Photo download: [Flickr](https://flic.kr/p/T6saUn)"
  #image:
#  feature: /assets/images/home-banner.jpg
#  thumb: /assets/logo.png
---
I seem to spin up a lot of Virtual Lab's and to make sure my lab doesn't interfere with the rest of my network and to simulate a larger enterprise environment you'd use a router.

I normally use pfSense as my virtual router of choice but decided recently to create a router on Windows Server 2016 Core.

This isn't the first time I'e installed or used Windows Server Core, in a previous lab ive used it for a DC but i've not had much experience with it and this is was the first time i've used Windows Server for a router let alone on Core.

Assumptions: 
- You've configured your Virtual Switches with one connected to your host's network and at least one private\internal network
- You've created the virtual machine with the associated NIC's attached
- You've installed Windows Server 2016 Core selecting the none GUI option

# Setting up the server
1. On first boot you'll be presented with a prompt asking to configure the Administrator password
2. We now need to go into a powershell session

```cmd
Powershell.exe
```

3. Next we need to name the computer

```Powershell
Rename-Computer -Name GW
```
1. Get-NetIPConfig
1. Rename-NetAdapter -Name Ethernet -NewName External
1. Rename-NetAdapter -Name "Ethernet 2" -NewName Internal
1. New-NetIPAddress -InterfaceAlias "Internal" -IPAddress 172.0.0.1 -PrefixLength 24
1. Set-DnsClientServerAddress -InterfaceAlias Internal -ServerAddresses 172.0.0.10, 192.168.1.1
1. Test-NetConnection
1. Restart-Computer
1. Set-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)" -Enabl
ed True
1. Install-WindowsFeature Routing, RSAT-RemoteAccess-PowerShell
1. Restart-Computer
1. Install-RemoteAccess -VpnType Vpn

exit out of powershell here and enter NETSH

1. netsh routing ip nat add interface External
1. netsh routing ip nat set interface External mode=full
1. netsh routing ip nat add interface Internal

If you want to disable ipv6
Disable-NetAdaptorBinding -InterfaceAlias Ethernet -computerID ms


Thanks to:
http://deploymentresearch.com/Research/Post/387/Install-a-Virtual-Router-based-on-Windows-Server-2012-R2-using-PowerShell
http://deploymentresearch.com/Research/Post/285/Using-a-virtual-router-for-your-lab-and-test-environment
https://www.howtogeek.com/112660/how-to-change-your-ip-address-using-powershell/
https://blogs.technet.microsoft.com/bruce_adamczak/2013/01/15/2012-core-survival-guide/







## Hyper-V Swich
1. Create an External switch connected to the External Network 
2. Create an Internal switch on a Priavate network

## Create Virtual Machine
1. Specify the name: LAB01-GW
2. Specify the Generation: 2
3. Assign Memory: 1024 + Use Dynamic Memory
4. Configure Network to External (we'll add a second adator later)
5. Create a new virtual hard drive and set the size to 60GB
6. Select install an os from bootable image file
7. finish
8. Open settings and Add Hardware > Network Adaptor and set to Internal

## Install Windows 2016 Core
Power the machine on and follow the wizard making sure you pick the 'Windows Server Standard Evaluation' not the desktop experience adetion.
