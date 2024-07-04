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
#Install VM Tools
#Rename Computer
$newComputerName = 'dc01' #FIXME: the computername 

$PackageName = "managementagent-9.3.3-x64" #FIXME: name depends from the version of the VM Tools
$InstallerType = "msi"

$LogApp = "C:\Windows\Temp\CitrixHypervisor-9.3.3.log"

#region Install VM Tools
#the assumption is that there is only one iso mounted, hence one drive
$opticalDriveLetter = (Get-CimInstance Win32_LogicalDisk | Where-Object {$_.DriveType -eq 5}).DeviceID
Get-ChildItem -Path $opticalDriveLetter
#$Source = "$PackageName" + "." + "$InstallerType"
$UnattendedArgs = "/i $(Join-Path -Path $opticalDriveLetter -ChildPath $($PackageName,$InstallerType -join '.')) ALLUSERS=1 /Lv $LogApp /quiet /norestart"

# should throw 0
(Start-Process msiexec.exe -ArgumentList $UnattendedArgs -Wait -Passthru).ExitCode

#Invoke-Item -Path $LogApp
#endregion

#region Rename Computer
Rename-Computer -NewName $newComputerName -Restart -Force
#endregion
```

### 3.2. AutomatedLab Module

Download the function Get-GitModule.ps1 which:

1. download module from Github - in this case the AutomatedLab
2. extract the archive to \appData\Local\Temp
3. copty the module folder to C:\Program Files\WindowsPowerShell\Modules\[moduleName]
4. remove the zip file - contains the full repo content
5. remove the extracted repository - contains the psm1

```powershell
# run in elevated PowerShell session
#region initialize variables
$scriptName     = 'Get-GitModule.ps1'
$uri            = 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode',$scriptName -join '/'
$path           = "$env:USERPROFILE\Documents"
$outFile        = Join-Path -Path $path -ChildPath $scriptName

$githubUserName = 'makeitcloudy'
$moduleName     = 'AutomatedLab'
#endregion

# download function Get-GitModule.ps1
Set-Location -Path $path
Invoke-WebRequest -Uri $uri -OutFile $outFile -Verbose
#psedit $outFile

#region run
# load function into memory
#. .\Get-GitModule
. $outFile

Get-GitModule -GithubUserName $githubUserName -ModuleName $moduleName -Verbose
#endregion

#removal of the function
Remove-Item -Path $outFile -Force -Verbose

# troubleshooting
Get-Module -Name $moduleName -ListAvailable
Get-Command -Module $moduleName

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