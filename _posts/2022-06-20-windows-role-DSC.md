---
full-width: true
layout: post
title: "DSC - Windows Role Setup"
permalink: "/windows-role-DSC/"
subtitle: "Setup Basic Windows Roles with Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up End User Computing infrastructure prerequisites, like ADCS, DHCP, File Services, SQL, by making use of PowerShell and Desired State Configuration, for home lab purposes.

## Links

[XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer

## 1. Windows Role Setup

### 1.1. Active Directory Certification Services

Active Directory Certification Services

#### 1.1.1. ADCS - Root CA

Run in (XCP-ng terminal over SSH). It deploys Active Directory Certificate Services - RootCA VM.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_adcsR' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:19' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_dhcp01_ADCS_RootCA_desktopExperience'

```

#### 1.1.2. ADCS - Sub CA

Run in (XCP-ng terminal over SSH). It deploys Active Directory Certificate Services - SubCA VM.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_adcsS' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:18' --StorageName 'node4_ssd_sdf' --VmDescription 'w2k22_dhcp02_ADCS_SubCA_desktopExperience'

```

#### 1.1.3. ADCS - VMTools installation

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_adcsR'
xe vm-cd-insert vm='c1_adcsR' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_adcsS'
xe vm-cd-insert vm='c1_adcsS' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run powershell code (elevated powershell session)

```powershell


```

Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_adcsR'
xe vm-cd-eject vm='c1_adcsS'

```


### 1.2. DHCP

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dhcp01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:11' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dhcp01_DHCPServer_core'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_dhcp02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:12' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dhcp02_DHCPServer_core'

```

#### 1.2.1. DHCP - VMTools installation

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_dhcp01'
xe vm-cd-insert vm='c1_dhcp01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_dhcp02'
xe vm-cd-insert vm='c1_dhcp02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run powershell code (elevated powershell session)

```powershell


```

Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_dhcp01'
xe vm-cd-eject vm='c1_dhcp02'

```

### 1.3. File Server

File Services

#### 1.3.1. File Server - ISCSI - Target

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_iscsi' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:09' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_iscsi_FileServer_desktop_experience'

# once the VM is installed add drives
# do not initialize them - do that from the failover cluster console

/opt/scripts/vm_add_disk.sh --vmName "c1_iscsi" --storageName "node4_hdd_sdc_lsi" --diskName "w2k22_c1_iscsi_quorumDrive" --deviceId 5 --diskGB 20  --description "w2k22_quorumDrive"
/opt/scripts/vm_add_disk.sh --vmName "c1_iscsi" --storageName "node4_hdd_sdc_lsi" --diskName "w2k22_c1_iscsi_vhdxClusterStorageDrive" --deviceId 6 --diskGB 100  --description "w2k22_vhdxClusterStorageDrive"

# add network interfaces to the VM
# * cluster network
# * storage network

```

#### 1.3.2. File Server - Member Server

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_fs01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:21' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_fs01_FileServer_core'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_fs02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:22' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_fs02_FileServer_core'

# once the VM is installed add drives
## Add Disk
/opt/scripts/vm_add_disk.sh --vmName 'fs01_core' --storageName 'node4_hdd_sdc_lsi' --diskName 'fs01_PDrive' --deviceId 4 --diskGB 60  --description 'fs01_ProfileDrive'
/opt/scripts/vm_add_disk.sh --vmName 'fs02_core' --storageName 'node4_hdd_sdc_lsi' --diskName 'fs02_PDrive' --deviceId 4 --diskGB 60  --description 'fs02_ProfileDrive'

```

#### 1.3.3. File Server - VMTools installation

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_iscsi'
xe vm-cd-insert vm='c1_iscsi' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_fs01'
xe vm-cd-insert vm='c1_fs01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_fs02'
xe vm-cd-insert vm='c1_fs02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run powershell code (elevated powershell session)

```powershell


```

Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_iscsi'

xe vm-cd-eject vm='c1_fs01'
xe vm-cd-eject vm='c1_fs02'

```

### 1.4. SQL

#### 1.4.1. SQL Server 2019

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_sql01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:31' --StorageName 'node4_ssd_sdf' --VmDescription 'w2k22_sql01_SQL2019_desktop_experience'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_sql02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:32' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_sql02_SQL2019_desktop_experience'
```

#### VMTools installation

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-insert vm='c1_sql01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_sql02'
xe vm-cd-insert vm='c1_sql02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run powershell code (elevated powershell session)

```powershell


```

Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-eject vm='c1_sql02'

```

## Windows Role - Configuration

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.10
