---
full-width: true
layout: post
title: "EUC lab - blog post series"
permalink: "/windows-euc-lab-blog-post-series/"
subtitle: "Container with details for the subsequent blog posts, focused on automated EUC oriented lab setup"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is a container with references to subsequent blog posts, which depicts the automated process of prerequisites installation for the EUC oriented home lab deployment.

**Note**:  

* [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - contains the code used to provision the VM's on the hypervisor layer
* The VM clipboard does not work over the XCP-ng console.

**Goals**:  

* .

**ToDo**:  

* describe the code references and dependecies  

# EUC lab - Blog post series

The series goes through the automated installation of the Windows Roles, which are essential for each End User Computing (EUC) lab deployment. Regardless if it is Citrix, Remote Desktop Services, Parallels RAS, each of those have depencies on the following roles.    
Here the amount of clicks in the GUI is eliminated to the bare minimum, and majority of the configuration is written in the code.  

[Active Directory Domain Services setup](https://makeitcloudy.pl/windows-role-active-directory/) - ADDS  
[Active Directory Certificate Services setup](https://makeitcloudy.pl/windows-role-active-directory-certificate-services/) - ADCS  
[DHCP Server setup](https://makeitcloudy.pl/windows-role-dhcp/) - DHCP  
[iSCSI Target File Server setup](https://makeitcloudy.pl/windows-role-fileServer-iSCSI-target/)  - iSCSI Target  
[File Server setup](https://makeitcloudy.pl/windows-role-fileServer-member/) - File Server Member  
[SQL Server setup](https://makeitcloudy.pl/windows-server-sql/) - SQL Server  

##

Current section focuses on documenting the code used in the blog posts mentioned above.  

## Code Details - ADDS

[ADDS](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#active-directory-domain-services) - GitHub code  

* [005_ActiveDirectory_demo.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1) script, downloads the [ADDS_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_setup.ps1) to the user profile documents directory.  
* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1)
* [InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfig.ps1)

### Code Details - ADDS_setup.ps1

* It contains the DSC script, DSC Configuration is included within the script
* The configuration data is completelly separated from the [ConfigData.psd1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) - this information is important, especially with the IP address modifications of the Domain Controllers, once those are changed, it should be also reflected in the ConfigData.psd1. Otherwise machines won't join to the domain, when the [scenario - Domain Joined VM](https://makeitcloudy.pl/windows-DSC/) from paragraph 2.2 is run

Detailed explanation of the steps to prepare target node (regardless if it is a management or active directory node) is available in the two blog posts

* [windows-preparation](https://makeitcloudy.pl/windows-preparation/) - paragraph 2.0.2
* [windows-dsc](https://makeitcloudy.pl/windows-DSC/) - paragraph

## Code Details - ADCS

[ADCS](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#adcs) - GitHub code  

```code
.
```

## Code Details - DHCP

[DHCP](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#dhcp) - GitHub code  

```code
.
```

## Code Details - File Services

[File Member Server]() - GitHub code

```code
.
```

### Code Details - File Services - iscsi

[iSCSI Target Server](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---1x-file-server---iscsi-target---desktop-experience) - GitHub code

```code
.
```

### Code Details - File Services - member server

[File Server Member](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-file-server---core) - GitHub code

```code
.
```

### Code Details - SQL 

[SQL Server](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md#windows---server-os---2x-sql-server---desktop-experience) - GitHub code

```code
.
```

## Summary

Last update: 2024.08.14
