---
layout: single
title: Creating a Gateway Router on Windows Server 2016 Core
#excerpt: "Because"
tags: [Windows Server 2016, Server Core, Powershell, Routing, Gateway]
date: 2017-05-09 00:00:00
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
I seem to spin up a lot of Virtual Labs. To make sure my lab doesn't interfere with the rest of my network, and to simulate a larger enterprise environment you should use a virtual router.

I normally use pfSense as my virtual router of choice, but decided recently to create a router on Windows Server 2016 Core.

This isn't the first time I've installed or used Windows Server Core - in a previous lab I have used it for a DC but I haven't had much experience with it, so this is was the first time I've used Windows Server for a router let alone on Core.

Assumptions: 
- You've configured your Virtual Switches with one connected to your host's network and at least one private\internal network
- You've created the virtual machine with the associated NICs attached
- You've installed Windows Server 2016 Core selecting the none GUI option

## Setting up the server

1. On first boot you'll be presented with a prompt asking to configure the Administrator password.

   ![FirstBoot](/assets/images/gatewayonhypervcore/firstboot.jpg)

1. We'll be doing most of the work in Powershell so we need to launch it.

   ```cmd
   > Powershell.exe
   ```
   ![FirstBoot](/assets/images/GatewayOnHyperVCore/Powershell.jpg)

1. First lets name the computer (ignore the prompt about rebooting, we'll do this after configuring the machine).

   ```Powershell
   > Rename-Computer -Name GW
   ```

1. We now want to rename the network adaptors, but to do this, we first need to find out the current names. Use the output from this to double check the MAC addresses with the NICs inside your virtualization software.

   ```Powershell
   > Get-NetIPConfiguration
   ```
   ![GetNetIPConfig](/assets/images/GatewayOnHyperVCore/GetNetIPConfig.jpg)
      
1. We then want to rename the adaptors using Rename-NetAdaptor. Using the -Name switch to pass the current names that we found in the previous step. Then use Get-NetIPConfiguration again to confirm.

   ```Powershell
   > Rename-NetAdapter -Name Ethernet -NewName External
   > Rename-NetAdapter -Name "Ethernet 2" -NewName Internal
   > Get-NetIPConfiguration
   ```
   ![RenameNetAdaptor](/assets/images/GatewayOnHyperVCore/RenameNetAdaptor.jpg)

1. Next we'll configure and validate the internal network adaptors IP details, DNS Addresses, and disable IPv6 for both adaptors. I'm setting my DNS addresses to 172.0.0.10 as this will be my DC, and 192.168.1.254 as this is my external router. 

   ```Powershell
   > New-NetIPAddress -InterfaceAlias Internal -IPAddress 172.0.0.1 -PrefixLength 24
   > Set-DnsClientServerAddress -InterfaceAlias Internal -ServerAddresses 172.0.0.10, 192.168.1.1
   > Disable-NetAdaptorBinding -Name Internal, External -ComponentID ms_tcpip6
   > Get-NetAdaptorBinding -Name Internal, External -ComponentID ms_tcpip6
   > Get-NetIPConfiguration
   > Test-NetConnection
   ```
   ![SetAdaptorSettings](/assets/images/GatewayOnHyperVCore/SetAdaptorSettings.jpg)

1. The last step is to reboot the computer.

   ```Powershell
   > Restart-Computer
   ```

## Installing and configuring the Gateway

1. After boot, login, and launch Powershell.
1. First, we need to enable a firewall rule used by routing.

   ```Powershell
   > Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"
   ```
  
1. Next, we need to install the Routing Windows Feature, plus the management tools and then reboot the computer.

   ```Powershell
   > Install-WindowsFeature Routing -IncludeAllSubFeature -IncludeManagementTools
   > Restart-Computer
   ```
   ![InstallFeature](/assets/images/GatewayOnHyperVCore/InstallFeature.jpg)

1. Once rebooted, re-login and launch Powershell to install the router.

   ```Powershell
   > Install-RemoteAccess -VpnType Vpn
   ```
   ![InstallRemoteAccess](/assets/images/GatewayOnHyperVCore/InstallRemoreAccess.jpg)

1. We now need to enter a NETSH session.

   ```Powershell
   > NETSH
   ```
1. The final step is to add some routing rules, were going to add the two interfaces, and configure the external mode.

   ```NETSH
   > routing ip nat add interface External
   > routing ip nat set interface External mode=full
   > routing ip nat add interface Internal
   ```
   ![NETSH](/assets/images/GatewayOnHyperVCore/NETSH.jpg)

## Validation

We can validate the config by creating a second VM with or without a GUI. Configuring the IP address inside the 172.0.0.0/24 range with a default gateway of the GW we've just configured (172.0.0.1), and the DNS address of your external router. We then use the the Test-NetConnection Powershell command to confirm external access.


   ![ConfigInternet](/assets/images/GatewayOnHyperVCore/ConfirmInternet.jpg)

Thats it, you should have now configured a Virtual Router on Windows Server 2016 Core. Let me know how it goes!

Thanks to the [2012 Core Survival Guide](https://blogs.technet.microsoft.com/bruce_adamczak/2013/01/15/2012-core-survival-guide/) and Deployment Researches guides on [setting up a virtual router](http://deploymentresearch.com/Research/Post/285/Using-a-virtual-router-for-your-lab-and-test-environment) and [setting up a virtual router using powershell](http://deploymentresearch.com/Research/Post/387/Install-a-Virtual-Router-based-on-Windows-Server-2012-R2-using-PowerShell).