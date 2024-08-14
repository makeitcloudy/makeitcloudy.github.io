---
full-width: true
layout: post
title: "File Server setup"
permalink: "/windows-role-fileServer-member/"
subtitle: "Setup Windows File Server Member with PowerShell and Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up Windows File Server Member by making use of Powershell and Desired State Configuration, for home lab purposes.

**Note**:

* [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer
* The VM clipboard does not work over the XCP-ng console.

**Goals**:

**ToDo**:

# 1. File Server Member

File Server setup is split into two pieces. Current scenario assumes that there is no external FreeNas for the exposure of iSCSI target.  
iSCSI for the further clustering setup is exposed on the Windows VM.  
[XCPng-scenario-HomeLab](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-file-server---core) - Github code  

## 1.2. VM provisioning - File Server - Member Server

Run in (XCP-ng terminal over SSH).

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_fs01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:21' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_fs01_Filer_core'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_fs02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:22' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_fs02_Filer_core'

```

## 1.3. VMTools installation - File Server - Member Server

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_fs01'
xe vm-cd-insert vm='c1_fs01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_fs02'
xe vm-cd-insert vm='c1_fs02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *fs01* and *fs02* for VM names
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_fs01'
xe vm-cd-eject vm='c1_fs02'

```

## 1.4. VM provisioning - File Server - Member Server - Add disk

Once the VM is installed add drives.  
Run in (XCP-ng terminal over SSH).

```bash
/opt/scripts/vm_add_disk.sh --vmName 'c1_fs01' --storageName 'node4_hdd_sdc_lsi' --diskName 'c1_fs01_PDrive' --deviceId 4 --diskGB 60  --description 'fs01_ProfileDrive'
/opt/scripts/vm_add_disk.sh --vmName 'c1_fs02' --storageName 'node4_hdd_sdc_lsi' --diskName 'c1_fs02_PDrive' --deviceId 4 --diskGB 60  --description 'fs02_ProfileDrive'

```

## 1.5. VM DSC configuration - File Server - Member Server

Run [run_initialConfigDsc_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1) in the elevated powershell session (VM).  
SubCA is member of the domain.

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose

```

**Note:** If the DSC configuration progress stuck on checking for disk, then in XCP-ng verify if the disk added to the VM, has Status Connected. There is great chance it's disconnected.

## 1.6. VM initial configuration - File Server

```powershell
#setup cluster
```

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.14
