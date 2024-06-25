---
layout: post
title: "Windows DSC authoring plane"
permalink: "/windows-authoring-vm/"
subtitle: "Setup Desired State Configuration Authoring box"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-windows.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft]
categories: [HomeLab ,Microsoft]
---
This Windows 10 VM is used for authoring the Desired State Configuration configurations.

# ImageFactory - OSDBuilder Nodes

Goals:

* Eval images are updated by making use of Segura's OSDBuilder
* No need for the winRM connectivity (unless it is needed for the loopback execution of the Desired State Configuration)
* Initial Configuration orchestrated by making use of DSC

* The image update process takes long time
* There are two types of operating systems to service - desktop and servers
* There are separate OSDBuilder Nodes for server OS'es, and Desktop OS'es
* Those two nodes are w10 based VM's

## Links

[How To Create A Windows Unattended Installation Image Including VMware Tools](https://www.leewoodhouse.com/2023/02/26/how-to-create-a-windows-unattended-installation-image-including-vmware-tools/)

## 0. Prerequisites - ImageFactory Nodes - w10_OSD_D and w10_OSD_S

* PowerShell 5.x
* ADK

* extra disk: ISO and OSDBuilder results - O:\ drive


## 1. Provisioning

At this point it is assumed:

1. There is SSH connectivity to the XCP-ng box
2. the xcp-ng scripts (used to provision vm's) are stored in /opt/scripts/

```bash
# TESTED - succesfull execution - 2024.VI.14
# run over SSH
/opt/scripts/vm_create_uefi.sh --VmName 'w10_OSD_D' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_updt_2302_unattended_noprompt.iso' --IsoSRName 'node4_nfs' --NetworkName 'NIC0 - .5.x' --Mac '5E:16:3e:5d:1f:41' --StorageName 'node4_ssd_sdb' --VmDescription 'w10_OSD_for_desktopOSes'
```

```bash
# TESTED - succesfull execution - 2024.VI.14
# run over SSH
/opt/scripts/vm_create_uefi.sh --VmName 'w10_OSD_S' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_updt_2302_unattended_noprompt.iso' --IsoSRName 'node4_nfs' --NetworkName 'NIC0 - .5.x' --Mac '5E:16:3e:5d:1f:42' --StorageName 'node4_ssd_sdb' --VmDescription 'w10_OSD_for_serversOSes'
```

## 2. Configuration

* initial configuration is arranged by Desired State Configuration

### 2.1 Add extra disk

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName "w10_OSD_D" --storageName "node4_hdd_sdc_lsi" --diskName "w10_OSD_D_dataDrive" --deviceId 4 --diskGB 100  --description "w10_OSD_D_dataDrive"
```

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName "w10_OSD_S" --storageName "node4_hdd_sdc_lsi" --diskName "w10_OSD_S_dataDrive" --deviceId 4 --diskGB 100  --description "w10_OSD_S_dataDrive"
```

### 2.2 Initialize disk

Run this on the VM

```powershell

```

### 2.3 DsC - Initial Configuration

Initial Configuration 

```powershell
# O:\ISO\not_updated\w10\eval_center
# O:\ISO\not_updated\w10\mediaCreationTool
# O:\ISO\updated\w10
```

## 3. Tools

## 4. OSDBuilder Process

### 4.1 w10_OSD_D - OSDBuilder - Image Factory for Desktop Based OS

#### 4.1.1 Microsoft Eval Center

Download missing ISO's:

* [Windows10 Eval Center](https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise) - LTSC/Non-LTSC 22H2
* *English (United States) ISO - Enterprise LTSC downloads* leads to: '19044.1288.211006-0501.21h2_release_svc_refresh_CLIENT_LTSC_EVAL_x64FRE_en-us.iso' - checked on 2024.06
* *English (United States) ISO - Enterprise downloads* leads to: '' - check on 2024.06

```code
as of 2024.06
https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise

ISO - Enterprise downloads      = 19045.2006.220908-0225.22h2_release_svc_refresh_CLIENTENTERPRISEEVAL_OEMRET_x64FRE_en-us = 10_22H2
ISO - Enterprise LTSC downloads = 19044.1288.211006-0501.21h2_release_svc_refresh_CLIENT_LTSC_EVAL_x64FRE_en-us            = 10_21H1_LTSC
```

#### 4.2.2 MediaCreationTool

Download missing ISO's - MediaCreationTool

```powershell
# Tested on 2024.06
# download https://github.com/AveYo/MediaCreationTool.bat# 
# move it to the data drive on the authoring box - tool is downloading iso's into the current directory where it is located

# in order to download the w10/w11 iso rename the MediaCreationTool accordingly:
# enterpriseN iso 21H2 MediaCreationTool.bat
# enterpriseN iso 22H2 MediaCreationTool.bat
# enterpriseN iso 11_23H2 MediaCreationTool.bat

# for some reason it was not possible to download w10 22H2 - it throws an error

```

### 4.2 w10_OSD_D - OSDBuilder - Installation

Requirements:

1. Local admin rights on the vm
2. Predownloaded ISO files
3. PowerShell command prompt run with elevated admin rights
4. Windows ADK Installed
5. Adequate disk space on the separate drive for the OSBuilder operations

Notes:
OSDBuilder 23.2.21.1 | OSD 24.6.18.1 Supports:
* Windows 10 1607 - 21H2
* Windows 11 21H2 - 22H2
* Windows Server 2016 1607 - Windows Server 2022 21H1

```powershell
Set-ExecutionPolicy ByPass -Scope CurrentUser -Force
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
Install-Module OSDBuilder -Force
# OSDBuilder 23.2.21.1 | OSD 24.6.18.1
# store the OSDBuilder within the separate directly aside from your regular OS
Get-OSDBuilder -SetHome O:\OSDBuilder
Get-OSDBuilder -CreatePaths
Import-Module OSDBuilder

# Mount downloaded ISO
# To get the index of the version which is under your interests, execute
# https://osdbuilder.osdeploy.com/module/functions/import-osmedia
# Import-OSMedia -ShowInfo -Quick
Import-OSMedia -ShowInfo -Quick
```

## OSDBuilder

Run this elevated

### Option 1

```powershell
Set-ExecutionPolicy ByPass -Scope CurrentUser
Install-Module OSDBuilder
# store the OSDBuilder within the separate directly aside from your regular OS
Get-OSDBuilder -SetPath O:\OSDBuilder
Get-OSDBuilder -CreatePaths
# nuget along with the untrusted module will be installed from the PS Gallery
Import-Module OSDBuilder
# Download the OneDriveSetup package
# OSDBuilder\Content folder contains all different content which is downloaded like
# one drive, or unattended installation or start menu customization
# eventually instead of 'OneDriveSetup Production' 'OneDriveSetup Enterprise' switch can be used
Get-DownOSDBuilder -ContentDownload 'OneDriveSetup Production'
# Mount the ISO of the media, planed for an update
# for windows 10 iso's it can be done with making use of MediaCreationTool
# https://github.com/AveYo/MediaCreationTool.bat

# In order to get the index of the version which is under your interests, execute
# https://osdbuilder.osdeploy.com/module/functions/import-osmedia
# Import-OSMedia -ShowInfo -Quick

# ImageIndex for Windows Servers editions is the natural number <1-4>
# ImageIndex 1 - standard
# ImageIndex 2 - standard desktop experience
# ImageIndex 3 - datacenter
# ImageIndex 4 - datacenter desktop experience

# ImageIndex for Desktop Operating systems - w10 is the natural number as well
# ImageIndex 1: Windows 10 Education
# ImageIndex 2: Windows 10 Education N
# ImageIndex 3: Windows 10 Enterprise
# ImageIndex 4: Windows 10 Enterprise N
# ImageIndex 5: Windows 10 Pro
# ImageIndex 6: Windows 10 Pro N
# in case it can not enumerate the indexes within the ISO, it may mean that the ISO is corrupted, you need to download it again and start the mount and import again
Import-OSMedia -ImageIndex 1 -SkipGrid -Update -BuildNetFX
# or
Import-OSMedia -ImageName 'Windows 10 Enterprise' -ImageIndex 3 -SkipGrid -Update -BuildNetFX -Verbose

# In this usecase there are no features which are enabled within the image
# It is left as generic as possible, and in case feature or role is needed it's
# getting enabled with making use of DSC (Desired State Configuration) later on
# when managing the configuration drift or with some other mechanisms

# once the whole operations ends it's time to create the ISO
# this command has a dependency on the ADK and the availability of oscdimg.exe
New-OSBMediaISO -FullName 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 1909 18363.2274'
# at this point the updated iso file should be located here
# O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 21H2 19044.2604\ISO

# at this point continue with step number 5 mentioned below
# this section will be updated shortly (2023.02.17 - recalling the process in the lab)
```

### Option 2

```powershell
#region - opcja 1
$isoName = '10_21H1_LTSC.iso'
$isoFolder = 'O:\ISO\notUpdated\w10\evalCenter'
$isoSourcePath = Join-Path -Path $isoFolder -ChildPath $isoName

$imageName = 'Windows Server 2022 Datacenter Evaluation Desktop Experience x64 21H2 20348.587'
$taskName = '202406_w2k22_eval_desktopExperience_updates'

Mount-DiskImage -ImagePath $isoSourcePath -Verbose

# 2. Import iso
# ImageIndex 1: Windows 10 Enterprise LTSC Evaluation
# ImageIndex 2: Windows 10 Enterprise N LTSC Evaluation
Import-OSMedia -Index 2

Get-OSMedia -GridView -Verbose

#Update-OSMedia -Download -Execute
# OS Media does not have the latest WSUSXML (MS Updates)
# Use the following command before furnning New-OSBuild
Update-OSMedia -Name $imageName -Download -Execute

# Now that the OS Media is imported and updated, you must create a task that you’ll use to re-run the monthly updates with any features you enable or disable.
# Run this command to create a new task (or re-run a previously created task). 
# In my case, I’m planning to enable .NET 3.5 SP1 so I’ve created the task name as shown below. 
# Create the name that makes sense for you.
New-OSBuildTask -TaskName $taskName -EnableNetFX3


New-OSBuild -Download -Execute -ByTaskName $taskName
#pick the OSImport

New-OSBMediaISO
#New-OSBMediaISO -FullName "O:\OSDBuilder\OSBuilds\$windowsServer2022ReleaseName"
#pick OSBuild - (pushed up all updates, and enabled netfx)

#ISO should be stored in following directory:
Invoke-Item -Path "O:\OSDBuilder\OSBuilds\$imageName\ISO"

#endregion
```

### Option 3

```powershell
# Tested 2024.06

$isoName = '10_21H1_LTSC.iso'
$isoFolder = 'O:\ISO\notUpdated\w10\evalCenter'
$isoSourcePath = Join-Path -Path $isoFolder -ChildPath $isoName

$imageName = 'Windows 10 Enterprise x64 21H2 19044.4529'
$taskName = '202406_w10_updates'

Mount-DiskImage -ImagePath $isoSourcePath -Verbose

# 2. Import iso
# ImageIndex 1: Windows 10 Enterprise LTSC Evaluation
# ImageIndex 2: Windows 10 Enterprise N LTSC Evaluation
Import-OSMedia -Index 2

# Unmount Diskimage

Get-OSMedia -GridView -Verbose

#Update-OSMedia -Download -Execute
# OS Media does not have the latest WSUSXML (MS Updates)
# Use the following command before furnning New-OSBuild
Update-OSMedia -Name $imageName -Download -Execute

# Now that the OS Media is imported and updated, you must create a task that you’ll use to re-run the monthly updates with any features you enable or disable.
# Run this command to create a new task (or re-run a previously created task). 
# In my case, I’m planning to enable .NET 3.5 SP1 so I’ve created the task name as shown below. 
# Create the name that makes sense for you.
New-OSBuildTask -TaskName $taskName -EnableNetFX3

New-OSBuild -Download -Execute -ByTaskName $taskName
#pick the OSImport

# copy the autounatted.xml here
Invoke-Item -Path 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 21H2 19044.4529\OS'

Get-OSBuilds

New-OSBMediaISO -FullName 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 21H2 19044.4529'
#New-OSBMediaISO -FullName "O:\OSDBuilder\OSBuilds\$windowsServer2022ReleaseName"
#pick OSBuild - (pushed up all updates, and enabled netfx)

#ISO should be stored in following directory:
Invoke-Item -Path "O:\OSDBuilder\OSBuilds\$imageName\ISO"

#Copy the image to your ISO repository
```

### 4.2 w10_OSD_S - OSDBuilder - Image Factory for Server Based OS

#### 4.2.1 Microsoft Eval Center

Download Missing ISOs:

* [Windows Server 2022 Eval Center](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)

## OSDBuilder

This document contains the guide to follow to prepare the Windows Server Operating System

```powershell
## ADK
# https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install
# Install only Deployment Tools

## OSDBuilder

Set-ExecutionPolicy ByPass -Scope CurrentUser -Force
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
Install-Module OSDBuilder -Force
Install-Module OSDSUS -Force
# OSDBUilder 23.2.21.1 | OSD 24.6.18.1
# store the OSDBuilder within the separate directly aside from your regular OS
Get-OSDBuilder -SetHome O:\OSDBuilder
Get-OSDBuilder -CreatePaths
Import-MOdule OSDBuilder

# Mount downloaded ISO
Mount-DiskImage -ImagePath O:\ISO\notUpdated\w10\evalCenter\10_21H1_LTSC.iso

# To get the index of the version which is under your interests, execute
# https://osdbuilder.osdeploy.com/module/functions/import-osmedia
get-help Import-OSMedia

# ImageIndex 1: Windows Server 2022 Standard Evaluation
# ImageIndex 2: Windows Server 2022 Standard Evaluation (Desktop Experience)
# ImageIndex 3: Windows Server 2022 Datacenter Evaluation
# ImageIndex 4: Windows Server 2022 Datacenter Evaluation (Desktop Experience)

Import-OSMedia -ImageIntex 2
Import-OSMedia -ImageIndex 3 -SkipGrid -BuildNetFX #windows server
#1113058
# Mount e:\Sources\install.wim
# Mount Install.wim O:\OSDBuilder\Mount\os111310
# Media: Copy Operating System to O:\OSDBuilder\OSImport\windows Server 2022 Datacenter Evaluation x64 21H2 20348.587\OS
# OS: Backup Auto Extra Files to \WinPE\\AutoExtrafiles
# OS: Exporting Sessions.xml Path Inventory
# OS: export Inventory to O:\OSDBuilder\OSImport\windows Server 2022 Datacenter Evaluation x64 21H2 20348.587
# WinPe: Export WIMs to O:\OSDBuilder\OSImport\windows Server 2022 Datacenter Evaluation x64 21H2 20348.587\WinPE

## OSD is Dismounting WindowsImage
# OS: save Windows Image Content to O:\OSDBuilder\OSImport\windows Server 2022 Datacenter Evaluation x64 21H2 20348.587\info\Get-WindowsImageContent.txt
## WSUSXML (Microsoft Updates) Download
## O:\OSDBuilder\Updates\[Name of the .cab]

### WARNING: This OSMedia does not have the latest WSUSXML (Microsoft Updates)
### WARNING: Use the following command before running New-OSBuild
### WARNING: Update-OSMedia -Name 'Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587' -Download -Execute

# MEDIA: Copy Operating System to O:\OSDBuilder\OSBuilds\build2406181138
# WinPE: Mount WinPE.wim to O:\OSDBuilder\Mount\winpe2406181139
# WinPE: Mount WinRE.wim to O:\OSDBuilder\Mount\winre2406181139
# WinPE: Mount WinSE.wim to O:\OSDBuilder\Mount\setup2406181139

# https://www.infotechram.com/wp-content/uploads/2020/11/How-to-install-and-use-OSDBuilder.pdf
# page 5

New-OSBuildTask -TaskName w2k22_21H2_build
Get-OSMedia | Where-Object {($_.Name -match "Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587") -and ($_.Revision -eq 'OK') -and ($_.Updates -eq 'Update')} |
              Foreach {Update-OSMedia -Download -Execute -Name $_.Name}

# Review the content create in the O:\OSDBuilder\OSBuilds folder
Get-ChildItem -Path 'O:\OSDBuilder\OSBuilds'

### Keeping the OSDBuilder platform updated
# As MS release new  updates, the OSD Builder platform must be updated too.
# These are commands to have some fun

OSDBuilder -Update
Update-OSDSUS
Import-Module -Name OSDBuilder -Force
Import-Module -Name OSDSUS -Force
Get-OSMedia | Where-Object {($_.Name -match "Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587") -and ($_.Revision -eq 'OK') -and ($_.Updates -eq 'Update')} |
              Foreach {Update-OSMedia -Download -Execute -Name $_.Name}
New-OSBuildTask -ByTaskName w2k22_21H2_build -Execute

Get-OSBuilds

New-OSBMediaIso -FullName 'O:\OSDBuilder\OSBuilds\Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587'
# iso file prepared this way boots properly on the XCP-ng when the Advanced -> Boot Firmware is set to BIOS
# [with the Boot Firmware set to UEFI - it shows only EFI shell]

```

## Summary

Last Update: 2024.06.20
