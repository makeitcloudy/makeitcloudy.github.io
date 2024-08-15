---
full-width: true
layout: post
title: "SQL Server setup"
permalink: "/windows-server-sql/"
subtitle: "Setup SQL Server with PowerShell and Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up SQL Server by making use of Powershell and Desired State Configuration, for home lab purposes.

**Note**:

* [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer
* The VM clipboard does not work over the XCP-ng console.

**Goals**:

**ToDo**:

## Links

* [PlagueHO](https://github.com/PlagueHO/LabBuilder/blob/main/source/dsclibrary/MEMBER_SQLSERVER2016.DSC.ps1) - LabBuilder - DSC Configurations

# 1. SQL Server

[XCPng-scenario-HomeLab](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-sql-server---desktop-experience) - Github code  

## 1.1. VM provisioning - SQL Server 2019

Run in XCP-ng terminal over SSH.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_sql01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:31' --StorageName 'node4_ssd_sdf' --VmDescription 'w2k22_sql01_SQL2019'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_sql02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:32' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_sql02_SQL2019'
```

Once OS is installed, proceed further.

## 1.2. VMTools installation - SQL Server

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-insert vm='c1_sql01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_sql02'
xe vm-cd-insert vm='c1_sql02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

## 1.3. VM initial configuration - SQL Server

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *sql01* and *sql02* for the second VM
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-eject vm='c1_sql02'

```

The disks can not be added unless the PV drivers are in the OS.  
The PV drivers initialization needs a reboot right after the installation.  
The first VM reboot take place after the VMTools installation, thus until this point of time adding drives won't be succesfull.

## 1.4. VM provisioning - SQL Server - Add disk

```bash
/opt/scripts/vm_add_disk.sh --vmName "c1_sql01" --storageName "node4_ssd_sdd" --diskName "w2k22_sql01_Sdrive" --deviceId 5 --diskGB 30  --description "w2k22_Sdrive_SQLDBdrive"
/opt/scripts/vm_add_disk.sh --vmName "c1_sql02" --storageName "node4_ssd_sde" --diskName "w2k22_sql02_Sdrive" --deviceId 5 --diskGB 30  --description "w2k22_Sdrive_SQLDBdrive"
# in case the disk is added or substracted, such modification should be adjusted in ConfigureNode.ps1
# ConfigureNode.ps1 - has the DSC configuration which initialize disks
# it bases on ConfigData.psd1 for the drive letter, label, format etc

# details, which script runs which part : 
# https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-sql-server---desktop-experience

```

## 1.5. VM DSC configuration - SQL Server

Run [run_initialConfigDsc_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1) in the elevated powershell session (VM).  
SubCA is member of the domain.

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose

```

## 1.6. Install SQL 2019

Mount SQL Server 2019 installation media

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-insert vm='c1_sql01' cd-name='SQLServer2019-x64-ENU.iso'
xe vm-cd-eject vm='c1_sql02'
xe vm-cd-insert vm='c1_sql02' cd-name='SQLServer2019-x64-ENU.iso'

```

Run in elevated powershell session

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# run in elevated PowerShell session
Set-Location -Path "$env:USERPROFILE\Documents"

#$sqlDefaultInstanceConfiguration = 'SQL_defaultInstance_configuration.ps1'
#$sqlDefaultInstanceSetup = 'SQL_defaultInstance_setup.ps1'
Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/009_SQL/SQL_defaultInstance_configuration.ps1' -OutFile "$env:USERPROFILE\Documents\SQL_defaultInstance_configuration.ps1" -Verbose
Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/009_SQL/SQL_defaultInstance_setup.ps1' -OutFile "$env:USERPROFILE\Documents\SQL_defaultInstance_setup.ps1" -Verbose
#psedit "$env:USERPROFILE\Documents\SQL_defaultInstance_setup.ps1"
#psedit "$env:USERPROFILE\Documents\SQL_defaultInstance_configuration.ps1"

# it launches the process of SQL installation
.\SQL_defaultInstance_setup.ps1

```

Eject installation media

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-eject vm='c1_sql02'

```

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.14
