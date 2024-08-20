---
full-width: true
layout: post
title: "Windows management VM - Preparation"
permalink: "/windows-preparation/"
subtitle: "Semi-manual prerequisites preparation on management VM"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft ,DSC]
categories: [HomeLab, Microsoft, DSC]
---
This Windows VM (desktop or server OS) is used as a starting point, acting as management node for the MS landscape.
Only initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

* there is no Active Directory yet
* the VM is installed from regular ISO, which can be downloaded from Microsoft Evaluation Center ([Windows 10](https://www.microsoft.com/en-us/software-download/windows10ISO),[Windows Server 2022](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022))

**Note**:

* The configuration is split into sections within the paragraphs of the blog post
* Subsequent paragraphs of the blog post contains code which can be run on the VM
* Code from paragraphs 1 - 1.2 - need to be executed one by one - with manual approach (jumping between XCP-ng and VM)
* Code from paragraphs 2.1 - 2.3 - can be run in one go by making use of the [InitialConfig.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfig.ps1) script mentioned in paragraph 2. All the sections below agregated under single run

**Goals**:

* GUI based administration is conducted here
* Once the domain is provisioned, it becomes domain joned
* Initial Configuration conducted by Desired State Configuration

**ToDo**:

* Idempotency into the code which wraps the DSC Configuration and it's execution
* Idempotency with the VMtools installations - check if those are installed already, if so, skip the installation process
* WinRM: Configure the trusted host section with DSC - crucial for the scenario when the configuration is run from central node
* OS   : Focus only on the tooling itself and it's installation
* OS   : Find the registry keys which configures the Edge

## Links

Bunch of usefull links, used within the logic:

* [AutomatedXCP-ng](https://github.com/makeitcloudy/AutomatedXCP-ng/) - the PowerShell module, it contains bunch of functions which helps with XCP-ng administration, it is downloaded during the execution of the logic to the management node, which is provisioned within current blog post
* AutomatedXCP-ng [/opt/scripts/vm_create_bios.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_bios.sh) - it is assumed that the *vm_create* scripts are stored on the XCP-ng host already within the */opt/scripts* directory
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi_secureBoot.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi_secureBoot.sh)
* AutomatedXCP-ng [/opt/scripts/vm_add_disk.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_add_disk.sh)
* Offhours - [Using PowerShell 7 as a replacement for Windows PowerShell 5.1](https://oofhours.com/2024/06/27/using-powershell-7-as-a-replacement-for-windows-powershell-5-1/)

## 0. Assumptions

At this point it is assumed:

* XCP-ng: There is SSH connectivity to the XCP-ng node from the endpoint device
* XCP-ng: The XCP-ng scripts, mentioned in the Links paragraph (used to provision vm's) are stored on XCP-ng node in /opt/scripts/ folder
* XCP-ng: Citrix Hypervisor tools are available on the ISO SR repository (*/var/opt/xen/ISO_Store*  - custom local iso storage created during the XCPng setup) and (*/opt/xensource/packages/iso* - default iso storage with XCPng tools)
* Mikrotik: On your network device of choice create reservation for the DHCP address, until you decide to go with the static IP
* Target Node: there is one extra drive added to the VM (it keeps the tooling binaries, etc)
* Target Node: it happens that the disk added to the VM is not connected properly to the VM on the XCP-ng, if this is the case fix it
* Target Node: Once the VM is provisioned, run the powershell code in elevated session

```powershell
# run a regular PowerShell session, cmd > powershell
Start-Process PowerShell_ISE -Verb RunAs
```

## 1. VM Installation

In this section, one VM which plays management node role, is setup. Run the code below on XCP-ng over SSH. The original source of the code: [XCPng-scenario-HomeLab.md](https://github.com/makeitcloudy/HomeLab/blob/feature/001_Hypervisor/_code/XCPng-scenario-HomeLab.md) - section *Windows - Desktop OS - Initial Configuration - Management Node - w10mgmt*

```bash
# Run on XCP-ng - Desktop OS - management Node
/opt/scripts/vm_create_uefi.sh --VmName 'c1_w10mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:C1:00:19' --StorageName 'node4_ssd_sdg' --VmDescription 'c1_w10mgmt'

# Run on XCP-ng - Server OS - management Node
```

At this point VM is already installed. It is assumed at this stage that the Citrix VMTools ISO is available in the ISO SR

* Eject OS installation media, mount VM Tools
* Add extra disk

```bash
# Run on XCP-ng over SSH
# eject OS installation media
xe vm-cd-eject vm='c1_w10mgmt'
# .iso should be available in following location: 
# /var/opt/xen/ISO_Store      - custom local iso storage created during the XCPng setup
# /opt/xensource/packages/iso - default iso storage with XCPng tools
xe vm-cd-insert vm='c1_w10mgmt' cd-name='Citrix_Hypervisor_821_tools.iso'

# add extra disk
/opt/scripts/vm_add_disk.sh --vmName 'c1_w10mgmt' --storageName 'node4_hdd_sdc_lsi' --diskName 'c1_w10mgmt_dataDrive' --deviceId 4 --diskGB 20  --description 'w10_mgmt_dataDrive'
```

Once done

* login to the VM via XenOrchestra Console window, or any other way you have handy, and get it's IP address
* alternatively if you have a reservation for the mac address on your DHCP server, get the IP from there
* XenServer on the CLI does not have a chance to get to know the IP, as there are no VMTools installed yet

When the IP Address of the VM is known

* Login to the VM via RDP. XCP-ng/Xen Orchestra Console does NOT provide a clipboard yet. This is why you need the VM IP, to login there via RDP and copy the code from sections below. [Remote Desktop Manager](https://devolutions.net/remote-desktop-manager/) from Devolutions is a great fit. Still mstsc, [Remote Desktop Connection Manager](https://learn.microsoft.com/en-us/sysinternals/downloads/rdcman) will also work.
* Username and it's credentials has been specified in the unattended.xml file which was used to spin up the VM.
* When you are logged in for the first time you are asked *'Do you want to allow your PC to be discoverable by other PCs and devices in this network'* - pick whatever option you like. As you are in your home network, you can opt for yes.
* Do not Restart the VM yet. Collect the IP address and login via RDP.

### 1.2 Initialize disk

 Run [InitializeDisk.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initializeDisk.ps1). Code below equals to the one in the link to the Github repository.

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# Run in elevated powershell session

# Disk should be connected to the VM, otherwise code won't work

# https://www.itprotoday.com/powershell/use-powershell-to-initialize-a-disk-and-create-partitions
# Z: | GPT | data drive

$driveLetter = 'Z'
$fileSystemLabel = 'dataDisk'
Get-Disk | Select-Object Number, IsOffline
#Initialize-Disk -Number 1 -PartitionStype GPT

$rawDisk = Get-Disk | Where-Object {$_.PartitionStyle -eq 'Raw'}
$rawDisk | Initialize-Disk -PartitionStyle GPT
New-Partition -DiskNumber $rawDisk.DiskNumber -DriveLetter $driveLetter -UseMaximumSize
Format-Volume -DriveLetter $driveLetter -FileSystem NTFS
Get-Volume -DriveLetter $driveLetter | Set-Volume -NewFileSystemLabel $fileSystemLabel
Get-ChildItem -Path $($driveLetter,':' -join '')
Get-PSDrive
```

## 2. HowTo

When this point of the VM provisioning is reached, there are two approaches:

* The code can be run from each sections mentioned in the blogpost below, or it can one go
* For the latter execute [run_initialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1) in elevated powershell session. The code stored in this file equals the the code from the paragraph 2.0.2 - if done - skip the paragraps 2.1 and 3
* Now, the initial configuration is complete, further settings are described in the blogpost [Windows-DSC](https://makeitcloudy.pl/windows-DSC/)
* For the RSAT installation - proceed with the code from paragraph 4. The goal is that this node performs the role of Management Node, so RSAT Tools, Admin Center, etc are desirable 
* For any Subsequent Applications and tools (SSMS, vscode, git) - proceed with the instructions from paragraph 5

### 2.0.1 What does the code do

* It installs vmTools
* It enables WinRM on desktop OS
* It downloads Get-GitModule.ps1 function for the sake of downloading the modules mentioned below and then once it's there it removes is, as the Get-GitModule is available as part of the AutomatedLab, for later use, at this stage there is no module yet, right?
* It downloads [AutomatedLab](https://github.com/makeitcloudy/AutomatedLab) module - it contains a bunch of functions which are used in the overall automation
* It downloads [AutomatedXCPng](https://github.com/makeitcloudy/AutomatedXCPng) module - it brings some functionality to manage the Xen/XCP-ng by making use of PowerShell SDK.
* It sets the power plan on the VM to high performance
* It renames the VM with the name provided during the code execution (it queries for the NewNodeName during the script execution)
* It restarts the Node

#### Prerequisites

* DNS resolving public addresses
* The VMTools ISO is mounted to VM

### 2.0.2 run_initialsetup.ps1

Run in the elevated powershell session (VM).  

 * [run_InitialSetup.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialsetupps1) 
 
```powershell
#Start-Process PowerShell_ISE -Verb RunAs
# run in elevated PowerShell session
#region initialize variables
$scriptName     = 'InitialConfig.ps1'
$uri            = 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode',$scriptName -join '/'
$path           = "$env:USERPROFILE\Documents"
$outFile        = Join-Path -Path $path -ChildPath $scriptName

#endregion

# set the execution policy
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Bypass -Force

#region download function Get-GitModule.ps1
Set-Location -Path $path
Invoke-WebRequest -Uri $uri -OutFile $outFile -Verbose
#psedit $outFile

# load function into memory
. $outFile
#psedit $outfile
Set-InitialConfiguration -Verbose
#endregion

```

Script contains function [Set-InitialConfiguration](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfig.ps1) which leads to the github code of *InitialConfig.ps1*, it contains the code performing the steps listed in paragraph 2.0.1.

## 3. Eject media

Eject the VMTools installation media.

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm='c1_w10mgmt'
```

## 4. Management Tools

* RSAT
* Windows Admin Center
* IIS Management Service
* IIS Manager for Remote Administration 1.2

### 4.1. RSAT Tools

VM configuration is arranged by PowerShell and Desired State Configuration. Installation of RSAT tools. Run ISE as administrator.

```powershell
# run in elevated PowerShell session
# Start-Process PowerShell_ISE.exe -Verb RunAs

# RSAT tools on the Desktop OS can NOT be installed by making use of DSC - it throws an error

#Get-WindowsCapability -Name RSAT* -Online | Select-Object -Property DisplayName, Name, State
#Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online

#Get-WindowsCapability -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0* -Online | Add-WindowsCapability -Online

#Install-WindowsFeature RSAT-RDS-Licensing-Diagnosis-UI

Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.BitLocker.Recovery.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.CertificateServices.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.DHCP.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.Dns.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.FailoverCluster.Management.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.FileServices.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.IPAM.Client.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.LLDP.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.NetworkController.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.NetworkLoadBalancing.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.RemoteAccess.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.RemoteDesktop.Services.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.ServerManager.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.Shielded.VM.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.StorageMigrationService.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.StorageReplica.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.SystemInsights.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.VolumeActivation.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.WSUS.Tools~~~~0.0.1.0
```

### 4.2. IIS Management Service

```code
* Install Windows Feature -> Internet Information Services -> Web Management Tools -> IIS Management Service
```

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerManagementTools -All
```

### 4.3. IIS Manager for Remote Administration

```code
* Download IIS Manager for Remote Administration 1.2
Internet -> https://www.microsoft.com/en-us/download/details.aspx?id=41177
```

```powershell
# tested on 2024.08
Start-Process 'https://www.microsoft.com/en-us/download/details.aspx?id=41177'
Invoke-WebRequest -Uri 'https://download.microsoft.com/download/2/4/3/24374C5F-95A3-41D5-B1DF-34D98FF610A3/inetmgr_amd64_en-US.msi'
```

## 5. Software Installation

Copy the software binaries to the Z: drive. Then have them installed.

* [AutomatedLab](https://github.com/makeitcloudy/AutomatedLab) module - at this stage it should be downloaded already to the C:\Program Files\WindowsPowerShell\Modules
* [AutomatedXCP-ng](https://github.com/makeitcloudy/AutomatedXCPng) module - already downloaded to the C:\Program Files\WindowsPowerShell\Modules, by the code run in previous paragraphs
* PowerShell 5.x
* [PowerShell 7.x](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows)
* XenServer PowerShell module
* [ImgBurn](https://www.imgburn.com/index.php?act=download)
* [Filezilla](https://filezilla-project.org/download.php)
* [Git](https://git-scm.com/downloads)
* [Visual Studio Code](https://code.visualstudio.com/download)
* [SSMS](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) - SQL Management Studio, [SMSS](https://learn.microsoft.com/en-us/sql/ssms/release-notes-ssms?view=sql-server-ver16#previous-ssms-releases) - previous releases

### 5.1. PowerShell 7.X

PowerShell 7.x is NOT needed for the initial configuration of the mgmt VM, provided 'PSDscResources' is used. If the 'PSDesiredStateConfiguration' is there, the problems starts to arise. Unless you stick with Modules and Resources going hand in hand with PSVersion 5.1.19041.236 - it's ok.

```powershell
# https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.4
# https://github.com/PowerShell/PowerShell/releases/download/v7.4.3/PowerShell-7.4.3-win-x64.msi
```

### 5.2. ImgBurn

It is used to prepare an ISO which contains XenTools. Download it from [ImgBurn download](https://www.imgburn.com/index.php?act=download).

```powershell
# once the ImgBurn is installed -> create image file from files/folders
# drag into the source folder:
#
# managementagent-9.3.3-x86.msi
# managementagent-9.3.3-x64.msi
# CitrixVMTools-Linux-8.2.1-1.tar.gz
#
# destination: Citrix_Hypervisor_821_tools.iso
```

### 5.3. FileZilla

It is used to copy the content to the Storage Repository of XCP-ng node.[FileZilla download](https://filezilla-project.org/download.php?type=client)

```powershell
# /opt/xensource/packages - xcp-ng - guest-tools-8.2.0 are located in this directory
# /var/opt/xen/ISO_Store - upload iso here

# disconnect from the server
```

Rescan the Storage Repository. It will include the uploaded ISO.

```shell
# Run on XCP-ng
xe sr-list name-label="node4_hdd_LocalISO"
xe sr-scan uuid="UUID of the abovementioned SR"
```

### 5.4. Git

Used to synchronize the code with the remote branches.

### 5.5 Visual studio code

Used to code.

```powershell
# https://code.visualstudio.com/docs/?dv=win64user
# extensions:
# * powershell
```

### 5.6. SQL Management Studio

Installation of SQL Management Studio

* [Install SQL Management Studio with DSC](https://www.sqlservercentral.com/articles/install-ssms-using-powershell-dsc)
* [Install SQL Management Studio](https://qawithexperts.com/article/sql/download-and-install-sql-server-management-studio-step-by-st/441)
* [Install SQL Server](https://qawithexperts.com/article/sql/download-and-install-sql-server-step-by-step-procedure/311)

```powershell
# https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms
# https://aka.ms/ssmsfullsetup

$media_path = "Z:\tools\SSMS_20_1\SSMS-Setup-ENU.exe"
$install_path = "$env:SystemDrive\SSMSto"
$params = "/Install /Quiet SSMSInstallRoot=`"$install_path`""

Start-Process -FilePath $media_path -ArgumentList $params -Wait

```

Once done SQL Server Management Studio 20 should arise in the start menu.

## 6. Download Prerequisites

Login to [https://citrix.com/account](https://citrix.com/account) with myCitrix credentials.

### 6.1. Citrix Hypervisor SDK

Download Citrix Hypervisor/XenServer SDK

```powershell
# 1. Login to citrix.com
# 2. Switch to my account
# 3. Download XenServer 8.2.3 LTSR SDK - https://www.citrix.com/downloads/citrix-hypervisor/product-software/hypervisor-82-premium-edition-CU1.html | 
# 3. Unblock the downloaded ZIP file - if you do not unblock the zip file you will end up with an error about something going wrong with the .dll file once importing module, once unblocked everything is fine
# 4. Once unblocked - Extract Zip file

# From the extracted zip copy the XenServerPSModule from the CitrixHypervisor-SDK\XenServerPowerShell folder to 

# C:\Program Files\WindowsPowerShell\Modules directory or
# $env:UserProfile\Documents\WindowsPowerShell\Modules for per-user configuration or 
# $env:windir\system32\WindowsPowerShell\v1.0\Modules  for system-wide configuration

# run new powershell session elevated - so it reloads the copied modules
```

### 6.2. VM Tools, XenCenter

Download:

* XenServer VM Tools for Windows
* XenCenter 2024.2.0 Windows Management Console

```powershell
# hamburger menu in top right corner -> Downloads -> 
# -> Citrix Hypervisor -> Citrix Hypervisor 8.2 LTSR CU1

# download the tools - as of 2024.06 - version: 9.3.3
```

## Summary

It was tested on:

* Windows 10 (22H2 - 19045.4529)
* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.05
