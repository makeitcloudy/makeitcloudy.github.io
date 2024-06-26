---
layout: post
title: "Windows management plane"
permalink: "/windows-mgmt-vm/"
subtitle: "Setup windows management box"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft]
categories: [HomeLab ,Microsoft]
---
This Windows 10 VM is used as a starting point, acting as management node for the MS landscape. Only initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

# MGMT Node - w10_mgmt node

* there is no Active Directory yet
* AD will be configured by making use of DSC

Goals:

* GUI based administration ise conducted here
* Once the domain is provisioned, w10_mgmt becomes domain joned
* WinRM is ENABLED
* Initial Configuration conducted by Desired State Configuration

ToDo:

* Separate the DSC configuration
* Focus only on the tooling itself and it's installation
* OS   : Find the registry keys which configures the Edge
* WinRM: Configure the trusted host section with DSC

## 0. Links

* [AutomatedXCP-ng](https://github.com/makeitcloudy/AutomatedXCP-ng/)
* AutomatedXCP-ng [/opt/scripts/vm_create_bios.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_bios.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi_secureBoot.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi_secureBoot.sh)
* AutomatedXCP-ng [/opt/scripts/vm_add_disk.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_add_disk.sh)

## 1. Prerequisites

At this point it is assumed:

1. XCP-ng: There is SSH connectivity to the XCP-ng node from the endpoint device
2. XCP-ng: The xcp-ng scripts (used to provision vm's, mentioned above) are stored on XCP-ng node in /opt/scripts/ folder
3. XCP-ng: Citrix Hypervisor tools are available on the ISO SR repository
4. Following tools are installed on the w10_mgmt node

* PowerShell 5.x                - OK
* PowerShell 7.x
* Management Box - RSAT tooling - OK
* SQL Management Studio         - OK
* XenServer PowerShell module   - OK
* AutomatedXCP-ng module        - OK
* Visual Studio Code            - 
* Git                           - 
* Filezilla                     - OK

## 2. Install VM

Run on XCP-ng

```bash
# Run on XCP-ng
/opt/scripts/vm_create_uefi.sh --VmName '_w10_mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:00:49' --StorageName 'node4_ssd_sdg' --VmDescription 'w10_mgmt_node'
```

* Right after the setup OS is asking you: Do you want to allow your PC to be discoverable by other PCs and devices on this network ? - NO
* XenServer PV Storage Host Adapter needs to restart the system to complete installation. - YES

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm=_w10_mgmt
```

## 3. Initial Configuration

Initial configuration is arranged by Desired State Configuration

```powershell
Start-Process PowerShell_ISE.exe -Verb RunAs

Get-ExecutionPolicy
# double check if this is a best practice
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force
```

### 3.1 Download Prerequisites

Login to [https://citrix.com/account](https://citrix.com/account) with myCitrix credentials.

#### 3.1.1 Citrix stack

Download:
* XenServer VM Tools for Windows
* XenCenter 2024.2.0 Windows Management Console

```powershell
# hamburger menu in top right corner -> Downloads -> 
# -> Citrix Hypervisor -> Citrix Hypervisor 8.2 LTSR CU1

# download the tools - as of 2024.06 - version: 9.3.3
```

#### 3.1.2 ImgBurn

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

#### 3.1.3 FileZilla

It is used to copy the Citrix_Hypervisor_821_tools.iso to the XCP-ng node.[FileZilla download](https://filezilla-project.org/download.php?type=client)

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

### 3.2 Install VMTools

Mount the ISO, and proceed with the installation of the VMTools on the w10_mgmt VM. Reboot the VM (seems it needs to be rebooted twice).

```powershell
# at this stage on your network device of choice create a static reservation for the DHCP address, until you decide to go with the static IP
```

### 3.3 Extra disk

Add Data Disk to the VM.

#### 3.2.1 Add disk

Run code on XCP-ng

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName "_w10_mgmt" --storageName "node4_hdd_sdc_lsi" --diskName "w10_mgmt_dataDrive" --deviceId 4 --diskGB 20  --description "w10_mgmt_dataDrive"
```

#### 3.2.2 Initialize disk

Initialize the disk diskmgmt.msc

```powershell
# GPT
# Z:
# data drive
# TODO: code for the disk initialization
```

Copy the downloaded tools into the Z: drive

### 3.3 Visual studio code

```powershell
# https://code.visualstudio.com/docs/?dv=win64user
# extensions:
# * powershell
```

### 3.4 PowerShell 7.X

PowerShell 7.x is NOT needed for the initial configuration of the mgmt VM, provided 'PSDscResources' is used. If the 'PSDesiredStateConfiguration' is there, the problems starts to arise. Unless you stick with Modules and Resources going hand in hand with PSVersion 5.1.19041.236 - it's ok.

```powershell
# https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.4
# https://github.com/PowerShell/PowerShell/releases/download/v7.4.3/PowerShell-7.4.3-win-x64.msi
```

### 3.5 RSAT Tools

Installation of RSAT tools. Run ISE as administrator.

```powershell
# RSAT tools on the Desktop OS can NOT be installed by making use of DSC - it throws an error

#Get-WindowsCapability -Name RSAT* -Online | Select-Object -Property DisplayName, Name, State
#Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online

#Get-WindowsCapability -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0* -Online | Add-WindowsCapability -Online

#Install-WindowsFeature RSAT-RDS-Licensing-Diagnosis-UI

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

### 3.10 SQL Management Studio

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

## 4. PowerShell

Installation of:

* Citrix Hypervisor SDK
* AutomatedXCPng - PowerShell module to orchestrate XCP-ng / XenServer

```powershell
update-help
New-Item -Path "$env:USERPROFILE\Documents\DSC\w10_mgmt" -ItemType Directory
```

### 4.2 Citrix Hypervisor SDK

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

### 4.3 AutomatedXCPng

Download https://github.com/makeitcloudy/AutomatedXCP-ng | Extract Zip.

```powershell
# Copy XenPLModule folder from AutomatedXCP-ng to C:\Program Files\WindowsPowerShell\Modules
# follow the guidelines: https://github.com/makeitcloudy/AutomatedXCPng

# XenPLModule folder should be copied into this location
# C:\Program Files\WindowsPowerShell\Modules
```

## 5. WinRM

Preconfigure the winRM. Run code on the w10_mgmt VM.

```powershell
### run code on w10_mgmt
Set-NetConnectionProfile -NetworkCategory Private
Enable-PSRemoting
### confirm winRM is running
Get-service -name winRM
#Get-service -name winRM | Start-Service
Get-Item WSMan:\localhost\Client\TrustedHosts


### ToDo: this portion should be configured by DSC
$destVMIP = @('10.2.134.211','10.2.134.212')

$destVMIP.ForEach({
    Test-NetConnection $_ –Port 5985
    Test-WsMan $_
})

get-Item WSMan:\localhost\Client\TrustedHosts

$destVMIP.ForEach({
    Set-Item WSMan:\localhost\Client\TrustedHosts -Value $_ -Concatenate -Force
})

Enter-PSSession -ComputerName 10.2.134.211

#endregion

#######

#region option 2
#region - run this on the management box

Set-Item wsman:\localhost\client\TrustedHosts -Value 10.2.134.210 -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 10.2.134.211 -Concatenate

$credential = Get-Credential

Enter-PSSession -ComputerName 10.2.134.210 -Credential $credential
Enter-PSSession -ComputerName 10.2.134.211 -Credential $credential
#endregion
```

## 6.2 DSC

```powershell
Install-PackageProvider -Name Nuget -MinimumVersion 2.8.5.201 -Force
Install-Module -Name 'PSDesiredStateConfiguration' -Force -AllowClobber
```

## 6.2.1 DSC - initial configuration

Initial Configuration which prepares the Folder Structure

```powershell
# save it as a dsc_config_InitialSetup.ps1
# Z:\_repo
# Z:\_repo\scripts_ps
# Z:\_repo\scripts_shell
# Z:\_repo\software

configuration InitialSetup {

  param(
    [string]$repoFolder = 'Z:\_repo'
    [string]$toolsFolder = 'Z:\tools'
    [string]$repoScriptsPSFolder = 'Z:\_repo\scripts_ps'
    [string]$repoScriptsShellFolder = 'Z:\_repo\scripts_shell'
  )

    Import-DscResource -ModuleName PSDesiredStateConfiguration

          File Repo {
            Type = 'Directory'
            DestinationPath = $repoFolder
            Ensure = "Present"
          }

          File Tools {
            Type = 'Directory'
            DestinationPath = $toolsFolder
            Ensure = "Present"
          }

          File RepoScripts {
            Type = 'Directory'
            DestinationPath = $repoScriptsPSFolder
            DependsOn = '[File]Repo'
          }

          File RepoScripts {
            Type = 'Directory'
            DestinationPath = $$repoScriptsShellFolder
            DependsOn = '[File]Repo'
          }

          # does not work for Desktop OS, does it trick for Server OS
          #WindowsFeature RSAT {
          #  Ensure = 'Present'
          #  Name = "RSAT-ADDS"
          #}
}

    
# Compile the configuration file to a MOF format
InitialConfiguration

# Run the configuration on localhost
Start-DscConfiguration -Path .\InitialConfiguration -Wait -Force -Verbose
```

## Summary

Last update: 2024.06.25
