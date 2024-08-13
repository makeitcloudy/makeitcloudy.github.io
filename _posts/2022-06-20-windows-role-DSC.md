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

### 1.1. Active Directory Domain Services

#### 1.1.1. VM provisioning - ADDS

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

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *dc01* for the first domain controller, and *dc02* for the second
* VM should reboot now

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

It deploys the Active Directory Role.

* Login back to VM, run the code in elevated powershell session. 
* When the AD deployment is finished, proceed with the same steps on the second VM.

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# run in elevated PowerShell session
Set-Location -Path "$env:USERPROFILE\Documents"

Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1' -OutFile "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -Verbose
#psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"

# at this stage the computername is already renamed and it's name is : dc01
. "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -ComputerName $env:Computername

```

#### 1.1.4. Role setup - ADDS - second domain controller

It deploys the Active Directory Role. 

* Login to the VM, run the code in elevated powershell session.
* This VM is added to the existing domain, which consists of one domain controller

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# run in elevated PowerShell session
Set-Location -Path "$env:USERPROFILE\Documents"

Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1' -OutFile "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -Verbose
#psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"

# at this stage the computername is already renamed and it's name is : dc01
. "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -ComputerName $env:Computername

```

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

#### 1.1.7. Troubleshoot - ADDS

Those type of issues are much easier to troubleshoot on Windows Server with Desktop Experience. Never the less here are some paths where the code is stored

```powershell
psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"
psedit "$env:USERPROFILE\Documents\ADDS_setup.ps1"
psedit C:\dsc\config\localhost\ActiveDirectory\ADDS_configuration.ps1
```

That's it for the Active Directory Domain Services. At this point you can add your VM's to the domain. It should work provided your network and DNS configurations are correct.

### 1.2. Active Directory Certification Services

Active Directory Certification Services

#### 1.2.1. VM provisioning - ADCS - Root CA

Run in (XCP-ng terminal over SSH). It deploys Active Directory Certificate Services - RootCA VM.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_adcsR' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:19' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_dhcp01_ADCS_RootCA_desktopExperience'

```

#### 1.2.2. VM provisioning - ADCS - Sub CA

Run in (XCP-ng terminal over SSH). It deploys Active Directory Certificate Services - SubCA VM.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_adcsS' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:18' --StorageName 'node4_ssd_sdf' --VmDescription 'w2k22_dhcp02_ADCS_SubCA_desktopExperience'

```

#### 1.2.3. VMTools installation - ADCS

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_adcsR'
xe vm-cd-insert vm='c1_adcsR' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_adcsS'
xe vm-cd-insert vm='c1_adcsS' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *dc01* for the first domain controller, and *dc02* for the second
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_adcsR'
xe vm-cd-eject vm='c1_adcsS'

```

### 1.3. DHCP

#### 1.3.1. VM provisioning - DHCP

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_dhcp01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:11' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dhcp01_DHCPServer_core'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_dhcp02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:12' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dhcp02_DHCPServer_core'

```

#### 1.3.1. VMTools installation - DHCP

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_dhcp01'
xe vm-cd-insert vm='c1_dhcp01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_dhcp02'
xe vm-cd-insert vm='c1_dhcp02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *dc01* for the first domain controller, and *dc02* for the second
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_dhcp01'
xe vm-cd-eject vm='c1_dhcp02'

```

### 1.4. File Server

File Services

#### 1.4.1. VM provisioning - File Server - ISCSI - Target

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

#### 1.4.2. VM provisioning - File Server - Member Server

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_fs01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:21' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_fs01_FileServer_core'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_fs02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:22' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_fs02_FileServer_core'

# once the VM is installed add drives
## Add Disk
/opt/scripts/vm_add_disk.sh --vmName 'fs01_core' --storageName 'node4_hdd_sdc_lsi' --diskName 'fs01_PDrive' --deviceId 4 --diskGB 60  --description 'fs01_ProfileDrive'
/opt/scripts/vm_add_disk.sh --vmName 'fs02_core' --storageName 'node4_hdd_sdc_lsi' --diskName 'fs02_PDrive' --deviceId 4 --diskGB 60  --description 'fs02_ProfileDrive'

```

#### 1.4.3. VMTools installation - File Server

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_iscsi'
xe vm-cd-insert vm='c1_iscsi' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_fs01'
xe vm-cd-insert vm='c1_fs01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_fs02'
xe vm-cd-insert vm='c1_fs02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *dc01* for the first domain controller, and *dc02* for the second
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_iscsi'

xe vm-cd-eject vm='c1_fs01'
xe vm-cd-eject vm='c1_fs02'

```

### 1.5. SQL

#### 1.5.1. VM provisioning - SQL Server 2019

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_sql01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:31' --StorageName 'node4_ssd_sdf' --VmDescription 'w2k22_sql01_SQL2019_desktop_experience'

/opt/scripts/vm_create_uefi.sh --VmName 'c1_sql02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:32' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_sql02_SQL2019_desktop_experience'
```

#### 1.5.2. VMTools installation - SQL Server

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-insert vm='c1_sql01' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_sql02'
xe vm-cd-insert vm='c1_sql02' cd-name='Citrix_Hypervisor_821_tools.iso'

```

Run in the elevated powershell session (VM).

* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), when asked, put the *dc01* for the first domain controller, and *dc02* for the second
* VM should reboot now

Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_sql01'
xe vm-cd-eject vm='c1_sql02'

```

## Windows Role - Configuration



## Code Details 

### Code Details - ADDS

The [005_ActiveDirectory_demo.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1) script, downloads the [ADDS_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_setup.ps1) to the user profile documents directory.

#### Code Details - ADDS_setup.ps1

* It contains the DSC script, DSC Configuration is included within the script
* The configuration data is completelly separated from the [ConfigData.psd1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) - this information is important, especially with the IP address modifications of the Domain Controllers, once those are changed, it should be also reflected in the ConfigData.psd1. Otherwise machines won't join to the domain, when the [scenario - Domain Joined VM](https://makeitcloudy.pl/windows-DSC/) from paragraph 2.2 is run

Detailed explanation of the steps to prepare target node (regardless if it is a management or active directory node) is available in the two blog posts

* [windows-preparation](https://makeitcloudy.pl/windows-preparation/) - paragraph 2.0.2
* [windows-dsc](https://makeitcloudy.pl/windows-DSC/) - paragraph

### Code Details - ADCS

```code
.
```

### Code Details - DHCP

```code
.
```

### Code Details - File Services

```code
.
```

#### Code Details - File Services - iscsi

```code
.
```

#### Code Details - File Services - member server

```code
.
```

### Code Details - SQL 

```code
.
```

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.10
