---
layout: post
title: "DSC - Active Directory setup"
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
/opt/scripts/vm_create_uefi.sh --VmName 'dc01_core' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '5E:16:3e:5d:1f:01' --StorageName 'node4_ssd_sdd' --VmDescription 'w2k22_dc01_core'
```

### VM - Automatic provisioning - w2k22_dc02 - Windows Server 2022 Core

```bash
# TESTED - succesfull execution - 2024.VI.14
/opt/scripts/vm_create_uefi.sh --VmName 'dc02_core' --VCpu 4 --CoresPerSocket 2 --MemoryGB 2 --DiskGB 32 --ActivationExpiration 180 --TemplateName 'Windows Server 2022 (64-bit)' --IsoName 'w2k22dtc_2302_core_untd_nprmt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '5E:16:3e:5d:1f:02' --StorageName 'node4_ssd_sde' --VmDescription 'w2k22_dc02_core'
```

## X. Configuration

### X.X Install VM tools

Run in XCP-ng terminal

```shell
# eject installation media
xe vm-cd-eject vm=dc01_core
xe vm-cd-eject vm=dc02_core

# insert VM Tools ISO

```

Run in XenOrchestra, VM Console

```powershell
# install VM tools
# 15
# d:
.\management-9.3.3-x64.msi
# check install IO drivers now
```

Run in XCP-ng

```shell
# eject installation media
xe vm-cd-eject vm=dc01_core
xe vm-cd-eject vm=dc02_core
```

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
$mgmtNodeIP = '10.2.134.249'
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

## w10_mgmt node

WinRM connection works

```powershell
## WinRM - test

$AdminUsername                      = "administrator"
$AdminPassword                      = ConvertTo-SecureString "Password1$" -AsPlainText -Force
$AdminCredential                    = New-Object System.Management.Automation.PSCredential ($AdminUsername, $AdminPassword)

$dc01 = New-PSSession -ComputerName '10.2.134.201' -Name 'dc01_core' -Credential $AdminCredential

Invoke-Command -Session $dc01 -ScriptBlock {$env:computername}
Invoke-Command -Session $dc01 -ScriptBlock {ipconfig /all}


## DSC prereq modules

Invoke-Command {$ProgressPreference = 'silentlycontinue'} -Session $s
Invoke-Command {Install-Module ComputerManagementDSC,NetworkingDSC -Force} -Session $s
Invoke-Command {Get-Module -Name ComputerManagementDSC,NetworkingDsc -ListAvailable} -Session $s
Invoke-Command {Get-DscResource HostsFile,SMBShare} -session $s | Select-Object Name,ModuleName,Version
```

## Summary

Last update: 2024.06.25,
