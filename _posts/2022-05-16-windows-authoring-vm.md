---
layout: post
title: "Windows DSC authoring VM"
permalink: "/windows-authoring-vm/"
subtitle: "Setup Desired State Configuration Authoring box"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft]
categories: [HomeLab ,Microsoft]
---
This Windows 10 VM is used for authoring the Desired State Configuration configurations.

# Authoring Node - w10_authoring node

* Here the Desired state configuration is prepared and invoked.
* The IP address of this node should be included 

## 0. Prerequisites

* PowerShell 5.x
* PowerShell 7.x
* Visual Studio Code
* Git
* Desired State Configuration Resources

* extra disk: DSC Configurations, Helper functions, powershell modules

* initial configuration arranged by Desired State Configuration

## 1. Provisioning

At this point it is assumed:

1. There is SSH connectivity to the XCP-ng box
2. the xcp-ng scripts (used to provision vm's) are stored in /opt/scripts/

```bash
/opt/scripts/vm_create.sh --VmName 'w10_mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_updt_2302_unattended_noprompt.iso' --IsoSRName 'node1_nfs' --NetworkName 'NIC0 - .5.x' --Mac '5E:16:3e:5d:1f:05' --StorageName 'node1_ssd_sdb' --VmDescription 'mgmtBox'
```

## 2. Configuration

### 2.1

```powershell
# 1. update powershell help

# 3. download ADK
# 
# 
```

### 2.2

```powershell
#1. initialize disk - GPT | O: drive | NTFS
#2. run ISE elevated
```

## Summary

Last update: 2024.06.25
