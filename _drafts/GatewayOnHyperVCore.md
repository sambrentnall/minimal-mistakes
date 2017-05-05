---
layout: single
title: Creating a Windows Server 2016 Core Gateway Router
#excerpt: "Because"
tags: [Windows Server 2016, Server Core, Powershell, Routing, Gateway]
date: 2017-05-04 00:00:00
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

## Setting up the server

1. On first boot you'll be presented with a prompt asking to configure the Administrator password

   ![FirstBoot](/assets/images/posts/GatewayOnHyperVCore/FirstBoot.jpg)

1. We'll be doing most of the work in Powershell so we need to launch it

   ```cmd
   > Powershell.exe
   ```
   ![FirstBoot](/assets/images/posts/GatewayOnHyperVCore/Powershell.jpg)

1. First things first, lets name the computer (ignore the prompt about rebooting, we'll do this after configuring the machine)

   ```Powershell
   > Rename-Computer -Name GW
   ```
1. We now want to rename the network adaptors but to do this, first we need to find out the current names, use this to double check the MAC addresses with the NIC's inside your virtualisation software

   ```Powershell
   > Get-NetIPConfiguration
   ```
   ![GetNetIPConfig](/assets/images/posts/GatewayOnHyperVCore/GetNetIPConfig.jpg)
      
1. We then want to rename the adaptors using Rename-NetAdaptor using the -Name switch to pass the current names that we found out in the previous step. Then use Get-NetIPConfiguration again to confirm

   ```Powershell
   > Rename-NetAdapter -Name Ethernet -NewName External
   > Rename-NetAdapter -Name "Ethernet 2" -NewName Internal
   > Get-NetIPConfiguration
   ```
   ![RenameNetAdaptor](/assets/images/posts/GatewayOnHyperVCore/RenameNetAdaptor.jpg)

1. Next we'll configure and validate the internal network adaptors IP details, DNS Addresses and disable IPv6 for both adaptors. I'm setting my DNS addresses to 172.0.0.10 as this will be my DC and 192.168.1.254 as this is my external router 

   ```Powershell
   > Sew-NetIPAddress -InterfaceAlias Internal -IPAddress 172.0.0.1 -PrefixLength 24
   > Set-DnsClientServerAddress -InterfaceAlias Internal -ServerAddresses 172.0.0.10, 192.168.1.1
   > Disable-NetAdaptorBinding -Name Internal, External -ComponentID ms_tcpip6
   > Get-NetAdaptorBinding -Name Internal, External -ComponentID ms_tcpip6
   > Get-NetIPConfiguration
   > Test-NetConnection
   ```
   ![SetAdaptorSettings](/assets/images/posts/GatewayOnHyperVCore/SetAdaptorSettings.jpg)
1. The last step is to reboot the computer

   ```Powershell
   > Restart-Computer
   ```

## Installing and configuring the Gateway


1. After boot, login and launch Powershell
1. First we need to enable a firewall rule used by routing

   ```Powershell
   > Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"
   ```
  
1. Next we need to install the Routing Windows Feature plus the management tools and reboot the computer

   ```Powershell
   > Install-WindowsFeature Routing -InludeAllSubFeature -IncludeManagementTools
   > Restart-Computer
   ```
   ![InstallFeature](/assets/images/posts/GatewayOnHyperVCore/InstallFeature.jpg)

1. Once rebooted, re-login and launch Powershell to install the router

   ```Powershell
   > Install-RemoteAccess -VpnType Vpn
   ```

exit out of powershell here and enter NETSH

1. netsh routing ip nat add interface External
2. netsh routing ip nat set interface External mode=full
3. netsh routing ip nat add interface Internal

If you want to disable ipv6
Disable-NetAdaptorBinding -InterfaceAlias Ethernet -ComponentID ms_tcpip6


Thanks to:
http://deploymentresearch.com/Research/Post/387/Install-a-Virtual-Router-based-on-Windows-Server-2012-R2-using-PowerShell
http://deploymentresearch.com/Research/Post/285/Using-a-virtual-router-for-your-lab-and-test-environment
https://www.howtogeek.com/112660/how-to-change-your-ip-address-using-powershell/
https://blogs.technet.microsoft.com/bruce_adamczak/2013/01/15/2012-core-survival-guide/