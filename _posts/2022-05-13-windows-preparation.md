---
layout: post
title: "Windows management VM - Preparation"
permalink: "/windows-preparation/"
subtitle: "Semi-manual prerequisites preparation on management VM"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft ,DSC]
categories: [HomeLab ,Microsoft ,DSC]
---
This Windows VM (desktop or server OS) is used as a starting point, acting as management node for the MS landscape.
Only initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

* there is no Active Directory yet
* the VM is installed from regular ISO, which can be downloaded from Microsoft Evaluation Center ([Windows 10](https://www.microsoft.com/en-us/software-download/windows10ISO),[Windows Server 2022](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022))

**Note**:

* The configuration is split into sections within the paragraphs of the blog post
* Subsequent paragraphs of the blog post contains code which can be run on the VM
* Code from paragraphs 1 - 1.2 - need to be executed one by one - with manual approach (jumping between XCP-ng and VM)
* Code from paragraphs 2.1 - 2.3 - can be run in one go by making use of the [windows-preparation.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/windows-preparation.ps1) script mentioned in paragraph 2. All the sections below agregated under single run

**Goals**:

* GUI based administration is conducted here
* Once the domain is provisioned, it becomes domain joned
* Initial Configuration conducted by Desired State Configuration

**ToDo**:

* Idempotency into the code which wraps the DSC Configuration and it's execution
* Idempotency with the VMtools installations - check if those are installed already, if so, skip the installation process
* *DSC  : Separate the Configuration, LCM into separate files - done*
* WinRM: Configure the trusted host section with DSC - crucial for the scenario when the configuration is run from central node
* OS   : Focus only on the tooling itself and it's installation
* OS   : Find the registry keys which configures the Edge

## Links

* [AutomatedXCP-ng](https://github.com/makeitcloudy/AutomatedXCP-ng/)
* AutomatedXCP-ng [/opt/scripts/vm_create_bios.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_bios.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi_secureBoot.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi_secureBoot.sh)
* AutomatedXCP-ng [/opt/scripts/vm_add_disk.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_add_disk.sh)

* Offhours - [Using PowerShell 7 as a replacement for Windows PowerShell 5.1](https://oofhours.com/2024/06/27/using-powershell-7-as-a-replacement-for-windows-powershell-5-1/)

## 0. Assumptions

At this point it is assumed:

* XCP-ng: There is SSH connectivity to the XCP-ng node from the endpoint device
* XCP-ng: The xcp-ng scripts (used to provision vm's, mentioned above) are stored on XCP-ng node in /opt/scripts/ folder
* XCP-ng: Citrix Hypervisor tools are available on the ISO SR repository
* On your network device of choice create a static reservation for the DHCP address, until you decide to go with the static IP
* .
* Code is run in elevated powershell session

```powershell
# run a regular PowerShell session, cmd > powershell
Start-Process PowerShell_ISE -Verb RunAs
```

### 0.1 Assumptions - Additional Disk Disk

* there is only one drive added to the VM
* the Data disk attached is connected on XCPng

### 0.2 Assumptions - XCP-ng

* the scripts are copied to /opt/scripts directory

## 1. VM Installation

Run on XCP-ng

```bash
# Run on XCP-ng - Desktop OS - management Node
/opt/scripts/vm_create_uefi.sh --VmName 'mgmtNode' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:00:49' --StorageName 'node4_ssd_sdg' --VmDescription 'mgmt_node'

# Run on XCP-ng - Server OS - management Node
```

Eject OS installation media

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm='mgmtNode'
```

Mount VMTools ISO

```bash
# run on XCP-ng
xe vm-cd-insert vm='mgmtNode' cd-name='Citrix_Hypervisor_821_tools.iso'
```

### 1.1 Add Data disk

Run on XCP-ng

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName 'mgmtNode' --storageName 'node4_hdd_sdc_lsi' --diskName 'mgmtNode_DesktopOS_dataDrive' --deviceId 4 --diskGB 20  --description 'mgmtNode_DesktopOS_dataDrive'
```

### 1.2 Initialize disk

Run in elevated powershell session

```powershell
#Start-Process PowerShell_ISE -Verb RunAs
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

## 2. All the sections below agregated under single run

When this point of the VM provisioning is reached, there are two approaches:

* the code can be executed from each sections mentioned below, or 
* the code can be run in one go, by making use of the code from the first section below:

```powershell
# run in elevated PowerShell session
#region initialize variables
$scriptName     = 'windows-preparation.ps1'
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
windows-preparation
#endregion
```

### 2.1 VMTools

1. Mount/Insert ISO
2. Proceed with the installation
3. Unmount/Eject ISO
3. Reboot VM (seems it needs to be rebooted twice).

#### 2.1.1 VMTools - installation code

Continue with this section and sections below 

```powershell
# run on VM (in elevated powershell session)
#Start-Process PowerShell_ISE -Verb RunAs

# https://support.citrix.com/article/CTX222533/install-xenserver-tools-silently
# https://forums.lawrencesystems.com/t/xcp-ng-installing-citrix-agent-for-windows-via-powershell-script/13855

$PackageName = 'managementagent-9.3.3-x64'
$InstallerType = 'msi'

$LogApp = 'C:\Windows\Temp\CitrixHypervisor-9.3.3.log'

$opticalDriveLetter = (Get-CimInstance Win32_LogicalDisk | Where-Object {$_.DriveType -eq 5}).DeviceID
Get-ChildItem -Path $opticalDriveLetter
#$Source = "$PackageName" + "." + "$InstallerType"
$UnattendedArgs = "/i $(Join-Path -Path $opticalDriveLetter -ChildPath $($PackageName,$InstallerType -join '.')) ALLUSERS=1 /Lv $LogApp /quiet /norestart"

# should throw 0
(Start-Process msiexec.exe -ArgumentList $UnattendedArgs -Wait -Passthru).ExitCode
#Invoke-Item -Path $LogApp

```

### 2.1.2 VMTools - eject media

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm='mgmtNode'
```

## 3. Prerequisites for the Desired State Configuration

### 3.1. Configure WinRM

WinRM should some attention on the Desktop OS. Regardless that piece of code should be run on Desktop OS and Server OS management nodes.

```powershell
#Rename-Computer -NewName 'w10mgmt' -Force -Restart
#$winRMServiceName = 'winRM' 

#check if CurrentUser is enough or LocalMachine is the correct one
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser -Force

# check if it is a desktop operating system
# in case it is then change the execution policy
$os = Get-CimInstance -ClassName Win32_OperatingSystem -ComputerName $ComputerName
switch($os.ProductType){
    '1' {
            Write-Output 'DesktopOS'
            #if((Get-Service -Name $winRMServiceName).Status -match 'Stopped'){
            #    Write-Warning "WinRM service is stopped"
            #    Start-Service -Name $winRMServiceName
            
            #region - NetConnectionProfile - set to private
            try {
                Write-Information 'Set NetConnectionProfile to Private'
                Set-NetConnectionProfile -NetworkCategory Private | Out-Null
                #Get-Item WSMan:\localhost\Client\TrustedHosts #empty
            }
            catch {

            }
            #endregion

            #region - Enable PS Remoting
            try {
                Enable-PSRemoting -Verbose
            }
            catch {

            }
            
            #endregion
        }
    '3' {
            Write-Output 'ServerOs'
            # PS Remoting seems to be configured already for the succesfull execution
        }
}

```

### 3.2. PowerShell Module - AutomatedLab - Download from Github

At this stage there are only default modules which are included in the operating system, so making use of 

* [Get-GiModule.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/Get-GitModule.ps1) - code
* [AutomatedLab](https://github.com/makeitcloudy/AutomatedLab) - Github repository

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

#region download function Get-GitModule.ps1
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

### 3.3. PowerShell Module - AutomatedXCPng - Download from Github

At this point AutomatedLab is already downloaded and extracted to PowerShell module repository, hence available commandlets. 

* [AutomatedLab](https://github.com/makeitcloudy/AutomatedLab) - Github repository
* [AutomatedXCPng](https://github.com/makeitcloudy/AutomatedXCPng) - Github repository

* AutomatedXCPng - PowerShell module to orchestrate XCP-ng / XenServer. For it to work properly it has a prerequisite of CitrixHypervisor SDK to be available on the system.

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

## 4. RSAT Tools

VM configuration is arranged by PowerShell and Desired State Configuration. Installation of RSAT tools. Run ISE as administrator.

```powershell
# run in elevated PowerShell session
# Start-Process PowerShell_ISE.exe -Verb RunAs

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

## 5. Download Prerequisites

Login to [https://citrix.com/account](https://citrix.com/account) with myCitrix credentials.

### 5.1. Citrix Hypervisor SDK

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

### 5.2. VM Tools, XenCenter

Download:

* XenServer VM Tools for Windows
* XenCenter 2024.2.0 Windows Management Console

```powershell
# hamburger menu in top right corner -> Downloads -> 
# -> Citrix Hypervisor -> Citrix Hypervisor 8.2 LTSR CU1

# download the tools - as of 2024.06 - version: 9.3.3
```

## 6. Software Installation

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

### 6.1. PowerShell 7.X

PowerShell 7.x is NOT needed for the initial configuration of the mgmt VM, provided 'PSDscResources' is used. If the 'PSDesiredStateConfiguration' is there, the problems starts to arise. Unless you stick with Modules and Resources going hand in hand with PSVersion 5.1.19041.236 - it's ok.

```powershell
# https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.4
# https://github.com/PowerShell/PowerShell/releases/download/v7.4.3/PowerShell-7.4.3-win-x64.msi
```

### 6.2. ImgBurn

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

### 6.3. FileZilla

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

### 6.4. Git

Used to synchronize the code with the remote branches.

### 6.5 Visual studio code

Used to code.

```powershell
# https://code.visualstudio.com/docs/?dv=win64user
# extensions:
# * powershell
```

### 6.6. SQL Management Studio

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

It was tested on: 

* Windows 10 (22H2 - 19045.4529)
* Server 2022 (21H2 - 20348.1547)

Last update: 2024.07.05
