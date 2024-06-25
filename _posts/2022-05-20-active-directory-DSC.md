---
layout: post
title: "Active Directory setup"
permalink: "/active-directory-DSC/"
subtitle: "Setup Active Directory with Desired State Configuration"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft]
categories: [HomeLab ,Microsoft]
---
This post is about setting up Active Directory by making use of Desired State Configuration, for home lab purposes.

# Domain Controllers

### VM - Automatic provisioning - domain controllers - Windows Server 2022 Core

This piece of code is executed via SSH directly on the XCP-ng node

```bash
# TESTED - succesfull execution - 2024.VI.14
/opt/scripts/vm_create_uefi.sh --VmName 'dc01_core' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 90 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_core_updt_2302_unattended_noprompt.iso' --IsoSRName 'node1_nfs' --NetworkName 'NIC0 - .5.x' --Mac '5E:16:3e:5d:1f:01' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dc01_core'
```

### VM - Automatic provisioning - w2k22_dc02 - Windows Server 2022 Core

```bash
# TESTED - succesfull execution - 2024.VI.14
/opt/scripts/vm_create_uefi.sh --VmName 'dc02_core' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_core_updt_2302_unattended_noprompt.iso' --IsoSRName 'node1_nfs' --NetworkName 'NIC0 - .5.x' --Mac '5E:16:3e:5d:1f:02' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dc02_core'
```

## X. Configuration

### X.X WinRM

```powershell
# https://woshub.com/using-psremoting-winrm-non-domain-workgroup/

#region server
# Make sure that the WinRM service is running on the target user’s computer:
Get-Service -Name "*WinRM*" | Select-Object status
# If the service is not running, enable it:
Enable-PSRemoting

Get-NetConnectionProfile
# if the network connection is set to public, need to change it to private
Set-NetConnectionProfile -NetworkCategory Private
# proof it's private
Get-NetConnectionProfile

# Open firewall
# The
$mgmtNodeIP = '10.2.134.250'
Set-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)" -RemoteAddress $mgmtNodeIP
Enable-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)"

# The WinRM HTTP Listener on the remote computer only allows connection with Kerberos authentication.
Get-ChildItem -Path WSMan:\localhost\Service\Auth\

# Type            Name                           SourceOfValue   Value
# ----            ----                           -------------   -----
# System.String   Basic                                          false
# System.String   Kerberos                                       true
# System.String   Negotiate                                      true
# System.String   Certificate                                    false
# System.String   CredSSP                                        false
# System.String   CbtHardeningLevel                              Relaxed

#endregion
```

## Summary

Last update: 2024.06.25,
