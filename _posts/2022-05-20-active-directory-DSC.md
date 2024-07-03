---
layout: post
title: "DSC - Active Directory setup"
permalink: "/active-directory-DSC/"
subtitle: "Setup Active Directory with Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft ,DSC]
categories: [HomeLab ,Microsoft, DSC]
---
This post is about setting up Active Directory by making use of Desired State Configuration, for home lab purposes.

# 0. Links

# 1. Assumptions

## 2. XCP-ng - VM provisioning

This piece of code is executed via SSH directly on the XCP-ng node

### 2.1 XCP-ng - first domain controller

```bash
# TESTED - succesfull execution - 2024.VI.14
/opt/scripts/vm_create_uefi.sh --VmName 'dc01' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '5E:16:3e:5d:1f:01' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dc01_core'
```

### 2.2 XCP-ng - second domain controller

```bash
# TESTED - succesfull execution - 2024.VI.14
/opt/scripts/vm_create_uefi.sh --VmName 'dc02' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '5E:16:3e:5d:1f:02' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dc02_core'
```

### 2.3 XCP-ng - eject installation media

Run once the VM's are provisioned

```shell
# eject installation media
xe vm-cd-eject vm=dc01
xe vm-cd-eject vm=dc02

# insert VM Tools ISO

```


## 3. Windows - prerequisites

## 3.1. Installation of VM tools

Install VM tools, rename computer

```powershell
# install VM tools
# 15
# d:
.\management-9.3.3-x64.msi
# check install IO drivers now
# at this stage install the tools silently
# do NOT reboot VM - it is rebooted during renaming
Rename-Computer -NewName dc01 -Restart -Force
```

### 3.2. AutomatedLab Module

```powershell
#region 0.2 - Zaimportowac Modul Automated Lab
# Pobrac folder z     : https://github.com/makeitcloudy/AutomatedLab/tree/feature/AutomatedLab
# Zapisac zawartosc do: C:\dsc\module\AutomatedLab
# Skopiowac do lokalizacji z modulami
# Zaimportowac Modul do sesji PS

#region TODO
# 1. zrobic modul AutomatedLab
# 2. zrzucic modul na Github
# 3. pobrac go lokalnie
# 4. zapisac / skopiowac do lokalizacji z modulami
# 5. zaladowac, tutaj tylko wywolac funkcje odpowiedzialna za instalacje modulow
# 6. nazwy wraz z wersjami modulow zdefiniowane jako parametr na gorze - przy inicjowaniu zmiennych

```

### 3.3. Desired State Configuration

Run on each VM acting as domain controller

```powershell
Set-Location -Path "$env:USERPROFILE\Documents"

Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1' -OutFile "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -Verbose
#psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"

# at this stage the computername is already renamed and it's name is : dc01
. "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1" -ComputerName $env:Computername
```


### Troubleshoot

```powershell
psedit "$env:USERPROFILE\Documents\ActiveDirectory_demo.ps1"
psedit "$env:USERPROFILE\Documents\ADDS_setup.ps1"
psedit C:\dsc\config\localhost\ActiveDirectory\ADDS_configuration.ps1
```

## XCP-ng - eject VM tools installation media

Run in XCP-ng

```shell
# eject installation media
xe vm-cd-eject vm=dc01_core
xe vm-cd-eject vm=dc02_core
```