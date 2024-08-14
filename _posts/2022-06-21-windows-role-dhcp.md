---
full-width: true
layout: post
title: "DHCP Server setup"
permalink: "/windows-role-dhcp/"
subtitle: "Setup Windows DHCP with PowerShell and Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up Windows DHCP Server by making use of Powershell and Desired State Configuration, for home lab purposes.

**Note**:

* [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer
* The VM clipboard does not work over the XCP-ng console.

**Goals**:

**ToDo**:

### 1.3. DHCP

[XCPng-scenario-HomeLab](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-dhcp-server---core) - Github code  

#### 1.3.1. VM provisioning - DHCP

Run in (XCP-ng terminal over SSH).

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dhcp01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:11' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dhcp01_DHCP_core'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_dhcp02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:12' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dhcp02_DHCP_core'

```

#### 1.3.2. VMTools installation - DHCP

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_dhcp01'
xe vm-cd-insert vm='c1_dhcp01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_dhcp02'
xe vm-cd-insert vm='c1_dhcp02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM). 

* Run [run_InitialSetup.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialsetupps1) in the elevated powershell session (VM).  
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_dhcp01'
xe vm-cd-eject vm='c1_dhcp02'

```

#### 1.3.5. VM DSC configuration - DHCP

Run [run_initialConfigDsc_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1) in the elevated powershell session (VM).  

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose

```

## 1.6. VM initial configuration - DHCP

```powershell
#setup cluster
```

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.14
