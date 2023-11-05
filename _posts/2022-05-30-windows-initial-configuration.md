---
layout: post
title: "Windows initial configuration"
permalink: "/windows-initial-configuration/"
subtitle: "Windows Server and Desktop OS"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.png
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, XCP-ng, Windows]
categories: [HomeLab, XCP-ng, Windows]
---
x

## Active Directory

Setup Windows server 2022 core edition, install management tools

```powershell
# count 180 days for the server operating system or 90 for the Desktop OS
# from the provisioning the VM on the hypervisor
(Get-Date).AddDays(180).ToString('yyyy-dd-MM')
```

## Azure AD Connect

Install Azure AD connector

```powershell
1. Navigate to https://portal.azure.com -> Azure Active Directory -> Azure AD connect
2. Download Microsoft Azure Active Directory Connect
# Azure Active Directory Connect - it does not support windows core
# Azure Active Directory Connect - it is not recommended to deploy on domain controllers
Start-Process 'https://www.microsoft.com/en-us/download/details.aspx?id=47594'
```

## Conclusions

Last update: 2022.05.25
