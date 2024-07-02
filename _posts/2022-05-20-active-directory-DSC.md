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

Run the following code on the target node.

It downloads the code to configure the winRM

```powershell
# 2024.06 - ToDo - comment the sections for the WSMan Get-ChildItem and Get-Item
# 2024.06 - ToDo - enable winRM only when the service is not running

$dscCodeRepoUrl        = 'https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_targetNode'
$initialSetup_FileName = 'initialSetup.ps1'
$initalsetup_ps1_url   = $dscCodeRepoUrl,$initialSetup_FileName  -join '/'
$outFile               = Join-Path -Path $env:USERPROFILE\Documents -ChildPath $initialSetup_FileName

Invoke-WebRequest -Uri $initalsetup_ps1_url -OutFile $outFile

Set-Location -Path $env:USERPROFILE\Documents
. .\initialSetup.ps1

Set-InitialConfiguration -MgmtNodeIPaddress 10.2.134.239 -Verbose

psedit $outFile

'https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_targetNode/initialSetup.ps1'

Set-Item wsman:localhost\client\trustedhosts -Value 10.2.134.239
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
Invoke-Command -Session $dc02 -ScriptBlock {$env:computername}


## DSC prereq modules

Invoke-Command {$ProgressPreference = 'silentlycontinue'} -Session $s
Invoke-Command {Install-Module ComputerManagementDSC,NetworkingDSC -Force} -Session $s
Invoke-Command {Get-Module -Name ComputerManagementDSC,NetworkingDsc -ListAvailable} -Session $s
Invoke-Command {Get-DscResource HostsFile,SMBShare} -session $s | Select-Object Name,ModuleName,Version
```

## Summary

Last update: 2024.06.25,
