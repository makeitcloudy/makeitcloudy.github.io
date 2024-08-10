---
full-width: true
layout: post
title: "DSC - Active Directory setup"
permalink: "/active-directory-DSC/"
subtitle: "Setup Active Directory with Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft ,DSC]
categories: [HomeLab ,Microsoft, DSC]
---
This post is about setting up Active Directory by making use of Desired State Configuration, for home lab purposes.

**Note**:

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

### VMtools

* [Setting automatic reboots when updating the Citrix VM Tools for Windows](https://support.citrix.com/s/article/CTX292687-setting-automatic-reboots-when-updating-the-citrix-vm-tools-for-windows?language=en_US)
* [Updates to XenServer VM Tools for Windows - For XenServer and Citrix Hypervisor](https://support.citrix.com/s/article/CTX235403-updates-to-xenserver-vm-tools-for-windows-for-xenserver-and-citrix-hypervisor?language=en_US)

### DSC Code

* [005_ActiveDirectory_demo.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1) - launcher of ADDS_setup.ps1
* [ADDS_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_setup.ps1) - ConfigData are here
* [ADDS_configuration.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_configuration.ps1) - DSC Configuration

## 1. VM Installation

In this section, two domain controller are being setup. Run the code below on XCP-ng over SSH.

```bash
# First domain controller - server core
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dc01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:01' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dc01_core'

# Second domain controller - server core
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dc02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:02' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dc02_core'
```

At this point VM is already installed. At this stage it is assumed, that the Citrix VMTools ISO is available in the ISO SR.

* Eject OS installation media, mount VM Tools

```bash
# Run on XCP-ng over SSH
# eject OS installation media
xe vm-cd-eject vm='c1_dc01'
# .iso should be available in following location: 
# /var/opt/xen/ISO_Store      - custom local iso storage created during the XCPng setup
# /opt/xensource/packages/iso - default iso storage with XCPng tools
xe vm-cd-insert vm='c1_dc01' cd-name='Citrix_Hypervisor_821_tools.iso'

# repeat the steps on second domain controller
xe vm-cd-eject vm='c1_dc02'
xe vm-cd-insert vm='c1_dc02' cd-name='Citrix_Hypervisor_821_tools.iso'
```

Once done

* login to the VM via XenOrchestra Console window, or any other way you have handy, and get it's IP address
* alternatively if you have a reservation for the mac address on your DHCP server, get the IP from there
* XenServer on the CLI does not have a chance to get to know the IP, as there are no VMTools installed yet

## 2. Howto

Detailed explanation of the steps is available in the two blog posts

* [windows-preparation](https://makeitcloudy.pl/windows-preparation/) - paragraph 2.0.2
* [windows-dsc](https://makeitcloudy.pl/windows-DSC/) - paragraph

Run following code on each Active Directory node.

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1)
* then once the machine is rebooted, run the code in elevated powershell session

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# run in elevated PowerShell session
Set-Location -Path "$env:USERPROFILE\Documents"

Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1' -OutFile "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -Verbose
#psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"

# at this stage the computername is already renamed and it's name is : dc01
. "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -ComputerName $env:Computername

```

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

