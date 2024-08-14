---
full-width: true
layout: post
title: "Active Directory Domain Services setup"
permalink: "/windows-role-active-directory/"
subtitle: "Setup Active Directory with PowerShell and Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up Active Directory by making use of Powershell and Desired State Configuration, for home lab purposes.

**Note**:
  
**DEPRECATED** - it was replaced by [Active Directory Domain Services Setup](https://makeitcloudy.pl/windows-role-active-directory/)

* The VM clipboard does not work over the XCP-ng console.
* The subnet where the node have the DHCP services enabled.
* Microsoft DHCP Server within the subnet is deployed later on.
* Then the DHCP Server within that subnet will be disabled.

```powershell
Get-NetConnectionProfile
# Name: Network
# InterfaceAlias: Ethernet 2
# InterfaceIndex: 7
# NetworkCategory: Public
Set-NetConnectionProfile -NetworkCategory Private | Out-Null
Set-NetFirewallRule -DisplayGroup "File and Printer Sharing" -Enabled True
```

**Goals**:

**ToDo**:

## Links

Bunch of usefull links

* VMware - Time - [Let’s Be Precise: Enabling and Configuring Precision Time Protocol in vSphere](https://blogs.vmware.com/apps/2021/04/lets-be-precise-enabling-and-configuring-precision-time-protocol-in-vsphere.html)
* VMware - Time - [Ensuring Accurate Time-Keeping in Virtualized Active Directory Infrastructure](https://blogs.vmware.com/apps/2020/09/ensuring-accurate-time-keeping-in-virtualized-active-directory-infrastructure.html)

### VMtools

* [Setting automatic reboots when updating the Citrix VM Tools for Windows](https://support.citrix.com/s/article/CTX292687-setting-automatic-reboots-when-updating-the-citrix-vm-tools-for-windows?language=en_US)
* [Updates to XenServer VM Tools for Windows - For XenServer and Citrix Hypervisor](https://support.citrix.com/s/article/CTX235403-updates-to-xenserver-vm-tools-for-windows-for-xenserver-and-citrix-hypervisor?language=en_US)

### DSC Code

The DSC, Configuration and Script are stored within the following links:

* [005_ActiveDirectory_demo.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1) - launcher of ADDS_setup.ps1
* [ADDS_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_setup.ps1) - ConfigData are here
* [ADDS_configuration.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_configuration.ps1) - DSC Configuration

### DSC and PowerShell code details

#### 2.1.1 ActiveDirectory_demo.ps1

The [005_ActiveDirectory_demo.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1) script, downloads the [ADDS_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_setup.ps1) to the user profile documents directory.

#### 2.1.1 ADDS_setup.ps1

* It contains the DSC script, DSC Configuration is included within the script
* The configuration data is completelly separated from the [ConfigData.psd1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) - this information is important, especially with the IP address modifications of the Domain Controllers, once those are changed, it should be also reflected in the ConfigData.psd1. Otherwise machines won't join to the domain, when the [scenario - Domain Joined VM](https://makeitcloudy.pl/windows-DSC/) from paragraph 2.2 is run

Detailed explanation of the steps to prepare target node (regardless if it is a management or active directory node) is available in the two blog posts

* [windows-preparation](https://makeitcloudy.pl/windows-preparation/) - paragraph 2.0.2
* [windows-dsc](https://makeitcloudy.pl/windows-DSC/) - paragraph

## 1. Windows Role Setup - Active Directory Domain Services

In this section, two domain controller are setup.

### 1.1. VM provisioning - ADDS

Run the code below on XCP-ng over SSH. 

The original source of the code: [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#active-directory-domain-services) - section *Windows - Server OS - 2x Domain Controller - Server Core*

```bash
# First domain controller - server core
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dc01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:01' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dc01_ADDS_core'

# Second domain controller - server core
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dc02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:02' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dc02_ADDS_core'

```

At this point:

* VM is already installed.
* It is assumed, that the Citrix VMTools ISO is available in the ISO SR.

#### 1.2. VMTools installation - ADDS

Eject OS installation media, mount VM Tools

Run in XCP-ng terminal over SSH.

```bash
# .iso should be available in following location: 
# /var/opt/xen/ISO_Store      - custom local iso storage created during the XCPng setup
# /opt/xensource/packages/iso - default iso storage with XCPng tools
xe vm-cd-eject vm='c1_dc01'
xe vm-cd-insert vm='c1_dc01' cd-name='Citrix_Hypervisor_821_tools.iso'
# repeat the steps on second domain controller
xe vm-cd-eject vm='c1_dc02'
xe vm-cd-insert vm='c1_dc02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

## 2. Role Setup - ADDS

For the quick run, on each Active Directory node, follow with the code mentioned below

1. [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *dc01* for the first domain controller, and *dc02* for the second
2. then once the first machine is rebooted, run the code in elevated powershell session. This piece of code deploy the Active Directory

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# run in elevated PowerShell session
Set-Location -Path "$env:USERPROFILE\Documents"

Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1' -OutFile "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -Verbose
#psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"

# at this stage the computername is already renamed and it's name is : dc01
. "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -ComputerName $env:Computername

```

3. when the AD deployment is finished, proceed with the same steps on the second VM (which become a second domain controller)

Run powershell code (elevated powershell session)

```powershell


```

Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_dc01'
xe vm-cd-eject vm='c1_dc02'

```

Now:

* login to the VM via XenOrchestra Console window, or any other way you have handy, and get it's IP address
* alternatively if you have a reservation for the mac address on your DHCP server, get the IP from there
* XenServer on the CLI does not have a chance to get to know the IP, as there are no VMTools installed yet

## 3. Active Directory - configuration

It's time for the Active Directory configuration (DNS, OU's, various objects).

The code below comes from the [Carl Webster - Building Websters's Lab v.2.1](https://www.carlwebster.com/building-websters-lab-v2-1/). Chapter 16 - Create Active Directory, contains the commandlets which are agregated in the [ADDS_CarlWebster_initialConfig.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_CarlWebster_initialConfig.ps1).

### 3.1. Active Directory - initial configuration

Run the code, on first domain controller: [ADDS_CarlWebster_initialConfig.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_CarlWebster_initialConfig.ps1) - it will be replicated across the controllers, without your intervention.

* It enables Recycle Bin
* It setup the password and lockout policy
* ~~It setup Sites~~
* It configures DNS scavenging and forwarders

### 3.2. Active Directory - structure

Run the code: [ADDS_structure.ps1]()
**TODO** - update the code link

* It  setup the AD structure (OU, groups, users)

# 4. Target VM

Now when the Active Directory domain is in place, add any VM spun meanwhile, to the ADDS. In order to do it, run the following code on the target VM:

```powershell
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose
```

It should result with the VM added to the domain.

**TODO** - check if the Set-InitialConfigDsc works after the VM reboot, not sure if the script should not be loaded into memory first...

Code caveats are described in the blog post [windows-DSC](https://makeitcloudy.pl/windows-DSC/), paragraph 3.

### Troubleshoot

Those type of issues are much easier to troubleshoot on Windows Server with Desktop Experience. Never the less here are some paths where the code is stored

```powershell
psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"
psedit "$env:USERPROFILE\Documents\ADDS_setup.ps1"
psedit C:\dsc\config\localhost\ActiveDirectory\ADDS_configuration.ps1
```

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.10

