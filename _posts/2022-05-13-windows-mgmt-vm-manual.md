---
layout: post
title: "Windows management VM - Manual setup"
permalink: "/windows-mgmt-vm-manual/"
subtitle: "A bit manual prerequisites preparation on management VM"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft ,DSC]
categories: [HomeLab ,Microsoft ,DSC]
---
This Windows VM (desktop or server OS) is used as a starting point, acting as management node for the MS landscape.
Only initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

* there is no Active Directory yet

Goals:

* GUI based administration ise conducted here
* Once the domain is provisioned, it becomes domain joned
* Initial Configuration conducted by Desired State Configuration

ToDo:

* Separate the DSC configuration
* Focus only on the tooling itself and it's installation
* OS   : Find the registry keys which configures the Edge
* WinRM: Configure the trusted host section with DSC

## Links

* [AutomatedXCP-ng](https://github.com/makeitcloudy/AutomatedXCP-ng/)
* AutomatedXCP-ng [/opt/scripts/vm_create_bios.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_bios.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi_secureBoot.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi_secureBoot.sh)
* AutomatedXCP-ng [/opt/scripts/vm_add_disk.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_add_disk.sh)

* Offhours - [Using PowerShell 7 as a replacement for Windows PowerShell 5.1](https://oofhours.com/2024/06/27/using-powershell-7-as-a-replacement-for-windows-powershell-5-1/)

## 0. Assumptions

At this point it is assumed:

1. XCP-ng: There is SSH connectivity to the XCP-ng node from the endpoint device
2. XCP-ng: The xcp-ng scripts (used to provision vm's, mentioned above) are stored on XCP-ng node in /opt/scripts/ folder
3. XCP-ng: Citrix Hypervisor tools are available on the ISO SR repository
4. On your network device of choice create a static reservation for the DHCP address, until you decide to go with the static IP

## 1. VM Installation

Run on XCP-ng

```bash
# Run on XCP-ng
/opt/scripts/vm_create_uefi.sh --VmName 'w10mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:00:49' --StorageName 'node4_ssd_sdg' --VmDescription 'w10_mgmt_node'
```

* Right after the setup OS is asking you: Do you want to allow your PC to be discoverable by other PCs and devices on this network ? - NO
* XenServer PV Storage Host Adapter needs to restart the system to complete installation. - YES

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm=w10-mgmt
```

### 1.1 Add Data disk

Run on XCP-ng

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName "w10mgmt" --storageName "node4_hdd_sdc_lsi" --diskName "w10_mgmt_dataDrive" --deviceId 4 --diskGB 20  --description "w10_mgmt_dataDrive"
```

### 1.2 Run an elevated powershell ISE instance

```powershell
# run PowerShell session
Start-Process PowerShell_ISE -Verb RunAs
```

### 1.3 Initialize disk

Run in elevated powershell session

####  Assumptions: 

* there is only one drive added to the VM
* the Data disk attached is connected

```powershell
# https://www.itprotoday.com/powershell/use-powershell-to-initialize-a-disk-and-create-partitions
# Z: | GPT | data drive

$driveLetter = 'Z'
Get-Disk | Select-Object Number, IsOffline
#Initialize-Disk -Number 1 -PartitionStype GPT

$rawDisk = Get-Disk | Where-Object {$_.PartitionStyle -eq ‘Raw’}
$rawDisk | Initialize-Disk -PartitionStyle GPT
New-Partition -DiskNumber $rawDisk.DiskNumber -DriveLetter $driveLetter -UseMaximumSize
Format-Volume -DriveLetter $driveLetter -FileSystem NTFS
Get-ChildItem -Path $($driveLetter,':' -join '')
Get-PSDrive
```

### 1.3 Install VMTools

VMTools installation.

1. Mount the ISO
2. Proceed with the installation
3. Rename the VM
3. Reboot the VM (seems it needs to be rebooted twice).

```bash
# run on XCP-ng
xe vm-cd-insert vm='w10mgmt' cd-name='Citrix_Hypervisor_821_tools.iso'
```

Install XenTools

```powershell
# https://support.citrix.com/article/CTX222533/install-xenserver-tools-silently
# https://forums.lawrencesystems.com/t/xcp-ng-installing-citrix-agent-for-windows-via-powershell-script/13855

# run on VM (in elevated powershell session)

$PackageName = "managementagent-9.3.3-x64"
$InstallerType = "msi"

$LogApp = "C:\Windows\Temp\CitrixHypervisor-9.3.3.log"

$opticalDriveLetter = (Get-CimInstance Win32_LogicalDisk | Where-Object {$_.DriveType -eq 5}).DeviceID
Get-ChildItem -Path $opticalDriveLetter
#$Source = "$PackageName" + "." + "$InstallerType"
$UnattendedArgs = "/i $(Join-Path -Path $opticalDriveLetter -ChildPath $($PackageName,$InstallerType -join '.')) ALLUSERS=1 /Lv $LogApp /quiet /norestart"

# should throw 0
(Start-Process msiexec.exe -ArgumentList $UnattendedArgs -Wait -Passthru).ExitCode

#Invoke-Item -Path $LogApp
```

Eject VMTools media

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm='w10mgmt'
```

Rename Computer

```powershell
#Rename-Computer -NewName 'w10mgmt' -Force -Restart
$winRMServiceName = 'winRM' 

#check if CurrentUser is enough or LocalMachine is the correct one
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser -Force

# check if it is a desktop operating system
# in case it is then change the execution policy
$os = Get-CimInstance -ClassName Win32_OperatingSystem -ComputerName $ComputerName
switch($os.ProductType){
    '1' {
        Write-Output 'DesktopOS'
        if((Get-Service -Name $winRMServiceName).Status -match 'Stopped'){
            Write-Warning "WinRM service is stopped"
            Start-Service -Name $winRMServiceName
        }
        }
    '3' {
        Write-Output 'ServerOs'
        }
}

```

### 1.4 AutomatedLab Module

[AutomatedLab](https://github.com/makeitcloudy/AutomatedLab)

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

# region download function Get-GitModule.ps1
Set-Location -Path $path
Invoke-WebRequest -Uri $uri -OutFile $outFile -Verbose
#psedit $outFile

# load function into memory
. $outFile
Get-GitModule -GithubUserName $githubUserName -ModuleName $moduleName -Verbose
#endregion

#removal of the function
Remove-Item -Path $outFile -Force -Verbose

# troubleshooting
#Get-Module -Name $moduleName -ListAvailable
#Get-Command -Module $moduleName
```

### 1.5 AutomatedXCPng Module

[AutomatedXCPng](https://github.com/makeitcloudy/AutomatedXCPng)

```powershell
# run in elevated PowerShell session
# follow the guidelines: https://github.com/makeitcloudy/AutomatedXCPng
# There prerequisite for the AutomatedXCPng to work properly is - Citrix Hypervisor Powershell Module / SDK

$githubUserName = 'makeitcloudy'
$moduleName     = 'AutomatedXCPng'

Get-GitModule -GithubUserName $githubUserName -ModuleName $moduleName -Verbose

# troubleshooting
#Get-Module -Name $moduleName -ListAvailable
#Get-Command -Module $moduleName
```

### 1.5 RSAT Tools

VM configuration is arranged by PowerShell and Desired State Configuration. Installation of RSAT tools. Run ISE as administrator.

```powershell
Start-Process PowerShell_ISE.exe -Verb RunAs

# RSAT tools on the Desktop OS can NOT be installed by making use of DSC - it throws an error

#Get-WindowsCapability -Name RSAT* -Online | Select-Object -Property DisplayName, Name, State
#Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online

#Get-WindowsCapability -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0* -Online | Add-WindowsCapability -Online

#Install-WindowsFeature RSAT-RDS-Licensing-Diagnosis-UI

Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0*
Add-WindowsCapability -Online -Name Rsat.FileServices.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.IPAM.Client.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.LLDP.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.NetworkController.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.NetworkLoadBalancing.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.BitLocker.Recovery.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.CertificateServices.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.DHCP.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.FailoverCluster.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.RemoteAccess.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.RemoteDesktop.Services.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.ServerManager.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.Shielded.VM.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.StorageMigrationService.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.StorageReplica.Tools~~~~0.0.1.0
Add-WindowsCapability -Online -Name Rsat.SystemInsights.Management.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.VolumeActivation.Tools~~~~0.0.1.0
#Add-WindowsCapability -Online -Name Rsat.WSUS.Tools~~~~0.0.1.0
```

## 2. Download Prerequisites

Login to [https://citrix.com/account](https://citrix.com/account) with myCitrix credentials.

### 2.2 Citrix Hypervisor SDK

Installation of:

* Citrix Hypervisor SDK
* AutomatedXCPng - PowerShell module to orchestrate XCP-ng / XenServer

### 2.3 Citrix Hypervisor SDK

Download XenServer SDK

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

### 2.4 VM Tools, XenCenter

Download:

* XenServer VM Tools for Windows
* XenCenter 2024.2.0 Windows Management Console

```powershell
# hamburger menu in top right corner -> Downloads -> 
# -> Citrix Hypervisor -> Citrix Hypervisor 8.2 LTSR CU1

# download the tools - as of 2024.06 - version: 9.3.3
```

## 3. Software Installation

Copy to the Z: drive.

* AutomatedLab module           - OK
* AutomatedXCP-ng module        - OK
* PowerShell 5.x                - OK
* PowerShell 7.x
* Management Box - RSAT tooling - OK
* XenServer PowerShell module   - OK
* ImgBurn                       - OK
* Filezilla                     - OK
* Git                           - 
* Visual Studio Code            - 
* SQL Management Studio         - OK

### 3.1 PowerShell 7.X

PowerShell 7.x is NOT needed for the initial configuration of the mgmt VM, provided 'PSDscResources' is used. If the 'PSDesiredStateConfiguration' is there, the problems starts to arise. Unless you stick with Modules and Resources going hand in hand with PSVersion 5.1.19041.236 - it's ok.

```powershell
# https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.4
# https://github.com/PowerShell/PowerShell/releases/download/v7.4.3/PowerShell-7.4.3-win-x64.msi
```

### 3.2 ImgBurn

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

### 3.3 FileZilla

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

### 3.4 Git

Used to synchronize the code with the remote branches.

### 3.5 Visual studio code

Used to code.

```powershell
# https://code.visualstudio.com/docs/?dv=win64user
# extensions:
# * powershell
```

### 3.6 SQL Management Studio

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

## Summary

Last update: 2024.06.25
