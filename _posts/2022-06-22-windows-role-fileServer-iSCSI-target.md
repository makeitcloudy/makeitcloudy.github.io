---
full-width: true
layout: post
title: "Windows iSCSI Target File Server setup"
permalink: "/windows-role-fileServer-iSCSI-target/"
subtitle: "Setup Windows iSCSI File Server with PowerShell and Desired State Configuration"
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

# 1. File iSCSI Target File Server

File Server setup is split into two pieces. Current scenario assumes that there is no external FreeNas for the exposure of iSCSI target.  
iSCSI for the further clustering setup is exposed on the Windows VM.  
[XCPng-scenario-HomeLab](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---1x-file-server---iscsi-target---desktop-experience) - Github code  

## 1.2. VM provisioning - File Server - iSCSI Target

Run in (XCP-ng terminal over SSH).

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_iscsi' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:09' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_iscsi_Filer'

```

## 1.3. VMTools installation - File Server - iSCSI Target

Run in XCP-ng terminal over SSH.

```bash

xe vm-cd-eject vm='c1_iscsi'
xe vm-cd-insert vm='c1_iscsi' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *iscsi* for VM name
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_iscsi'

```

## 1.4. VM provisioning - File Server - iSCSI Target - Add disk

Once the VM is installed add drives.  
Run in (XCP-ng terminal over SSH).

```bash
# do not initialize them - do that from the failover cluster console
/opt/scripts/vm_add_disk.sh --vmName 'c1_iscsi' --storageName 'node4_hdd_sdc_lsi' --diskName 'c1_iscsi_quorumDrive' --deviceId 5 --diskGB 20  --description 'w2k22_quorumDrive'
/opt/scripts/vm_add_disk.sh --vmName 'c1_iscsi' --storageName 'node4_hdd_sdc_lsi' --diskName 'c1_iscsi_vhdxClusterStorageDrive' --deviceId 6 --diskGB 100  --description 'w2k22_vhdxClusterStorageDrive'

```

add network interfaces to the VM:

* cluster network
* storage network

## 1.5. VM DSC configuration - File Server - iSCSI Target

Run [run_initialConfigDsc_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1) in the elevated powershell session (VM).  
SubCA is member of the domain.

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose

```

## 1.6. VM provisioning - File Server - iSCSI Target

```powershell
#setup ISCSI target
```