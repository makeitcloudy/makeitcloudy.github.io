---
full-width: true
layout: post
title: "Active Directory Domain Services setup"
permalink: "/windows-role-active-directory/"
subtitle: "Setup Windows Active Directory Domain Services with PowerShell and Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up Windows iSCSI Target File Server by making use of Powershell and Desired State Configuration, for home lab purposes.

**Note**:

* [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer
* The VM clipboard does not work over the XCP-ng console.

**Goals**:

**ToDo**:


# 1. Active Directory Domain Services

It deploys Active Directory Domain Services - Primary and Secondary Domain Controller VM.  
[XCPng-scenario-HomeLab](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-domain-controller---server-core) - Github code  

## 1.1. VM provisioning - ADDS

Run in (XCP-ng terminal over SSH). 

```bash
# First domain controller - server core
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dc01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:01' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dc01_ADDS_core'

# Second domain controller - server core
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dc02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:02' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dc02_ADDS_core'

```

At this point:

* VM is already installed.
* It is assumed, that the Citrix VMTools ISO is available in the ISO SR.

#### 1.1.2. VMTools installation - ADDS

Run in (XCP-ng terminal over SSH).

```bash
# it works - provided there is only one iso on SR with such name
# .iso should be available in following location: 
# /var/opt/xen/ISO_Store      - custom local iso storage created during the XCPng setup
# /opt/xensource/packages/iso - default iso storage with XCPng tools
xe vm-cd-eject vm='c1_dc01'
xe vm-cd-insert vm='c1_dc01' cd-name='Citrix_Hypervisor_821_tools.iso'
# repeat the steps on second domain controller
xe vm-cd-eject vm='c1_dc02'
xe vm-cd-insert vm='c1_dc02' cd-name='Citrix_Hypervisor_821_tools.iso'
```

#### 1.1.3. VM initial configuration

Run [run_InitialSetup.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialsetupps1) in the elevated powershell session (VM).  
When asked, put the *dc01* for the first domain controller, and *dc02* for the second

It goes through initial configuration of the VM.

* it install VM tools
* it downloads the AutomateLab module
* it downloads the AutomatedXCPng module
* it sets registry key as per CTX222533
* it sets the powershell execution policy
* it configures WinMR
* it set the power plan - high performance
* it renames the VM
* it reboots the VM

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_dc01'
xe vm-cd-eject vm='c1_dc02'

```

Now:

* login to the VM via XenOrchestra Console window, or any other way you have handy, and get it's IP address
* alternatively if you have a reservation for the mac address on your DHCP server, get the IP from there
* XenServer on the CLI does not have a chance to get to know the IP, as there are no VMTools installed yet

#### 1.1.3. Role setup - ADDS - first domain controller

Run [run_ADDS.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_addsps1) in the elevated powershell session (VM).  
It deploys the Active Directory Role.

* Login back to VM, run the code in elevated powershell session. 
* When the AD deployment is finished, proceed with the same steps on the second VM.

#### 1.1.4. Role setup - ADDS - second domain controller

Run [run_ADDS.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_addsps1) in the elevated powershell session (VM).  

It adds second VM to the existing Active Directory domain, which consists of one domain controller, so far. 

* Login to the VM, run the code in elevated powershell session.

#### 1.1.5. Initial Configuration - ADDS

It's time for the Active Directory configuration (DNS, OU's, various objects).  
The code below comes from the [Carl Webster - Building Websters's Lab v.2.1](https://www.carlwebster.com/building-websters-lab-v2-1/). Chapter 16 - Create Active Directory, contains the commandlets which are agregated in the [ADDS_CarlWebster_initialConfig.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_CarlWebster_initialConfig.ps1).  
Run the code, on first domain controller: [ADDS_CarlWebster_initialConfig.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_CarlWebster_initialConfig.ps1) - it will be replicated across the controllers, without your intervention.

* It enables Recycle Bin
* It setup the password and lockout policy
* ~~It setup Sites~~
* It configures DNS scavenging and forwarders

#### 1.1.6. Initial Logical Structure - ADDS

Run the code, on first domain controller: [ADDS_structure.ps1]()  
**TODO** - update the code link

* It setup the AD structure (OU, groups, users)

#### 1.1.7. Further steps - add existing VM to the domain

Now the management node which has been prepared in earlier [blog post](https://makeitcloudy.pl/windows-DSC/) , can be added to the existing domain.  
Run [run_initialConfigDsc_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1) in the elevated powershell session (VM).  

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose

```

#### 1.1.8. Troubleshoot - ADDS

Those type of issues are much easier to troubleshoot on Windows Server with Desktop Experience. Never the less here are some paths where the code is stored

```powershell
psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"
psedit "$env:USERPROFILE\Documents\ADDS_setup.ps1"
psedit C:\dsc\config\localhost\ActiveDirectory\ADDS_configuration.ps1
```

That's it for the Active Directory Domain Services. At this point you can add your VM's to the domain. It should work provided your network and DNS configurations are correct.

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.14
