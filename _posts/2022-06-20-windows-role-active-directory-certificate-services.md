---
full-width: true
layout: post
title: "Active Directory Certificate Services setup"
permalink: "/windows-role-active-directory-certificate-services/"
subtitle: "Setup Active Directory Certificate Services with PowerShell and Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up End User Computing infrastructure prerequisites, like ADCS, DHCP, File Services, SQL, by making use of PowerShell and Desired State Configuration, for home lab purposes.

## Links

* [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer
* [Building Webster's Lab V1](https://www.carlwebster.com/building-websters-lab-v1/) - 2019
* [Building Webster's Lab V2](https://www.carlwebster.com/building-websters-lab-v2/) - 2021
* [Building Webster’s Lab V2.1](https://www.carlwebster.com/building-websters-lab-v2-1/) - 2023

# 1. Active Directory Certification Services

Active Directory Certification Services  
[XCPng-scenario-HomeLab](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---1x-adcs-root-1x-adcs-sub---desktopexperience) - Github code  

## 1.2. VM provisioning - ADCS - Root CA

Run in (XCP-ng terminal over SSH). It deploys Active Directory Certificate Services - RootCA VM.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_adcsR' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:19' --StorageName 'node4_ssd_sdg' --VmDescription 'w2k22_dhcp01_ADCS_RootCA'

```

## 1.3. VM provisioning - ADCS - Sub CA

Run in (XCP-ng terminal over SSH). It deploys Active Directory Certificate Services - SubCA VM.

```bash
/opt/scripts/vm_create_uefi.sh --VmName 'c1_adcsS' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:18' --StorageName 'node4_ssd_sdf' --VmDescription 'w2k22_dhcp02_ADCS_SubCA'

```

## 1.4. VMTools installation - ADCS

Run in XCP-ng terminal over SSH.

```bash
xe vm-cd-eject vm='c1_adcsR'
xe vm-cd-insert vm='c1_adcsR' cd-name='Citrix_Hypervisor_821_tools.iso'

xe vm-cd-eject vm='c1_adcsS'
xe vm-cd-insert vm='c1_adcsS' cd-name='Citrix_Hypervisor_821_tools.iso'

```

## 1.5. VM initial configuration - ADCS

Run [run_InitialSetup.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialsetupps1) in the elevated powershell session (VM).  
Eject VMTools installation media. Run bash code (XCP-ng terminal over SSH)

```bash
xe vm-cd-eject vm='c1_adcsR'
xe vm-cd-eject vm='c1_adcsS'

```

Now:

* login to the VM via XenOrchestra Console window, or any other way you have handy, and get it's IP address
* alternatively if you have a reservation for the mac address on your DHCP server, get the IP from there
* XenServer on the CLI does not have a chance to get to know the IP, as there are no VMTools installed yet

## 1.6. VM DSC configuration - ADCS - RootCA

Run [run_initialConfigDsc_workgroup.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_workgroupps1) in the elevated powershell session (VM).  
RootCA is **not** a member of the domain.

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Workgroup -Verbose

```

## 1.7. VM DSC configuration - ADCS - SubCA

Run [run_initialConfigDsc_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1) in the elevated powershell session (VM).  
SubCA is member of the domain.

```powershell
#cmd
#powershell
#Start-Process PowerShell -Verb RunAs
$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose

```

## Summary

It was tested on:

* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.14
