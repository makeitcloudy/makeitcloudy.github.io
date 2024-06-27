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

## Links

* [AutomatedXCP-ng](https://github.com/makeitcloudy/AutomatedXCP-ng/)
* AutomatedXCP-ng [/opt/scripts/vm_create_bios.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_bios.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi.sh)
* AutomatedXCP-ng [/opt/scripts/vm_create_uefi_secureBoot.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_create_uefi_secureBoot.sh)
* AutomatedXCP-ng [/opt/scripts/vm_add_disk.sh](https://github.com/makeitcloudy/AutomatedXCP-ng/blob/main/bash/vm_add_disk.sh)

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
/opt/scripts/vm_create_uefi.sh --VmName '_w10_mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_2302_untd_nprmpt_uefi.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:00:49' --StorageName 'node4_ssd_sdg' --VmDescription 'w10_mgmt_node'
```

* Right after the setup OS is asking you: Do you want to allow your PC to be discoverable by other PCs and devices on this network ? - NO
* XenServer PV Storage Host Adapter needs to restart the system to complete installation. - YES

```bash
# Run on XCP-ng
# eject installation media
xe vm-cd-eject vm=_w10_mgmt
```

### 1.1 Add Data disk

Run on XCP-ng

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName "_w10_mgmt" --storageName "node4_hdd_sdc_lsi" --diskName "w10_mgmt_dataDrive" --deviceId 4 --diskGB 20  --description "w10_mgmt_dataDrive"
```

### 1.2 Initialize disk

Initialize the disk diskmgmt.msc

```powershell
# GPT
# Z:
# data drive
# TODO: code for the disk initialization
```

### 1.3 Install VMTools

Mount the ISO, and proceed with the installation of the VMTools on the w10_mgmt VM. Reboot the VM (seems it needs to be rebooted twice).

```powershell
#
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

### 2.5 AutomatedXCPng

Download https://github.com/makeitcloudy/AutomatedXCP-ng | Extract Zip.

```powershell
# Copy XenPLModule folder from AutomatedXCP-ng to C:\Program Files\WindowsPowerShell\Modules
# follow the guidelines: https://github.com/makeitcloudy/AutomatedXCPng

# XenPLModule folder should be copied into this location
# C:\Program Files\WindowsPowerShell\Modules
```

## 3. Software Installation

Copy to the Z: drive.

* PowerShell 5.x                - OK
* PowerShell 7.x
* Management Box - RSAT tooling - OK
* SQL Management Studio         - OK
* XenServer PowerShell module   - OK
* AutomatedXCP-ng module        - OK
* Visual Studio Code            - 
* Git                           - 
* Filezilla                     - OK

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

## 4. VM Configuration

VM configuration is arranged by PowerShell and Desired State Configuration.

```powershell
Start-Process PowerShell_ISE.exe -Verb RunAs
update-help
New-Item -Path "$env:USERPROFILE\Documents\DSC\w10_mgmt" -ItemType Directory

```

### 4.1 RSAT Tools

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

## 5. Desired State Configuration

```powershell
Get-ExecutionPolicy
# double check if this is a best practice
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force

```

### 5.1 DSC - Configuration files

Download from github and store in $env:USERPROFILE\Documents\dsc_config_w10mgmt\:

* [ConfigData.psd1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig/ConfigData.psd1)
* [ConfigureLCM.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig/ConfigureLCM.ps1)
*[ConfigureNode.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig/ConfigureNode.ps1)


```powershell
$env:USERPROFILE\Documents\dsc_config_w10mgmt\ConfigData.psd1
$env:USERPROFILE\Documents\dsc_config_w10mgmt\ConfigureLCM.ps1
$env:USERPROFILE\Documents\dsc_config_w10mgmt\ConfigureNode.ps1
```

#### 5.1.1 $env:USERPROFILE\Documents\000_w10mgmt_initialConfig.ps1

Create [000_w10mgmt_initialConfig.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig.ps1) in $env:USERPROFILE\Documents directory.

```powershell
$env:USERPROFILE\Documents\000_w10mgmt_initialConfig.ps1
```

It has the following content.

```powershell
#region w10mgmt - initial configuration
# file is bound with 000_w10mgmt_initialConfig
# $dscConfigPath - should be created on w10mgmt VM, files

# * ConfigurationData.psd1
# * ConfigureLCM.ps1
# * ConfigureNode.ps1

# stored in $dscConfigPath directory

# * w10mgmt_initialConfig_demo.ps1 

# stored in $env:USERPROFILE\Documents directory
# it contains commandlines for succesfull execution of the DSC Configuration stored in
# three files mentioned above

#region w10mgmt - initial configuration 
$env:COMPUTERNAME

#region w10mgmt - initial checks
update-help
Get-ExecutionPolicy
Get-Service -Name WinRM #stopped
Test-WSMan -ComputerName localhost #can not connect 
Get-Item WSMan:\localhost\Client\TrustedHosts #winRM is not running hence error during execution
#endregion

#region w10mgmt - change network interface name
### assumption - there is only one network interface 
### Get the current network interface with a name that is not "Ethernet"
#$tempNetworkInterfaceName = 'Eth'
#$networkInterface = Get-NetAdapter | Where-Object { $_.Name -ne 'Ethernet' }
#
### Check if a network interface was found and rename it to "Ethernet"
#if ($networkInterface) {
#    Rename-NetAdapter -Name $networkInterface.Name -NewName $tempNetworkInterfaceName
#} else {
#    Write-Output "No network interface found to rename."
#}
#endregion

#region WinRM configuration
Set-NetConnectionProfile -NetworkCategory Private
Enable-PSRemoting
Get-Item WSMan:\localhost\Client\TrustedHosts #empty
#endregion
#endregion

#region 001 - DSC - demo
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force

Install-PackageProvider -Name Nuget -MinimumVersion 2.8.5.201 -Force

# Seems 'PSDesiredStateConfiguration' module can not be installed otherwise it throws error during the LCM setup

# Import-Module : The version of Windows PowerShell on this computer is '5.1.19041.2364'. The module 'C:\Program Files\WindowsPowerShell\Modules\PSDesiredS
# tateConfiguration\2.0.7\PSDesiredStateConfiguration.psd1' requires a minimum Windows PowerShell version of '6.1' to run. Verify that you have the minimum
#  required version of Windows PowerShell installed, and then try again.
# At line:3 char:25
# + ...             Import-Module PSDesiredStateConfiguration -Verbose:$false ...
# +                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
#     + CategoryInfo          : ResourceUnavailable: (C:\Program File...figuration.psd1:String) [Import-Module], InvalidOperationException
#     + FullyQualifiedErrorId : Modules_InsufficientPowerShellVersion,Microsoft.PowerShell.Commands.ImportModuleCommand
#  
# PSDesiredStateConfiguration\Configuration : The module 'PSDesiredStateConfiguration' could not be loaded. For more information, run 'Import-Module PSDesi
# redStateConfiguration'.
# At C:\Users\labuser\Documents\dsc_config_w10mgmt\ConfigureLCM.ps1:2 char:1
# + Configuration ConfigureLCM {
# + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~
#     + CategoryInfo          : ObjectNotFound: (PSDesiredStateC...n\Configuration:String) [], CommandNotFoundException
#     + FullyQualifiedErrorId : CouldNotAutoLoadModule

#Install-Module -Name 'PSDesiredStateConfiguration' -Force -AllowClobber

#Get-Module -ListAvailable -Name 'PSDesiredStateConfiguration' | Uninstall-Module
#Get-Module -ListAvailable -Name 'PSDesiredStateConfiguration' | Remove-Module

#region - always run - initialize variables
#region initialize variables - DSC
$drive                              = 'Z:'
$dscConfigFolder                    = 'DSCConfigs'
# DSC mof files full path
$dscConfigFullPath                  = Join-Path -Path $drive -ChildPath $dscConfigFolder

# DSC configuration location
$dscConfigPath                      = "$env:USERPROFILE\Documents\dsc_config_w10mgmt\"
# DSC configuration data file - stored in the same directory as DSC configuration
$dscConfigDataFileName              = "ConfigData.psd1"
# DSC configuration data file full path
$dscConfigDataPath                  = Join-Path -Path $dscConfigPath -ChildPath $dscConfigDataFileName

# Provide credentials for DSC to use
$AdminUsername                      = "labuser"
$AdminPassword                      = ConvertTo-SecureString "Password1$" -AsPlainText -Force
$AdminCredential                    = New-Object System.Management.Automation.PSCredential ($AdminUsername, $AdminPassword)
#endregion

#region initialize variables - New-SelfSignedCertificateEx.ps1 function
$mypwd                               = ConvertTo-SecureString -String "Password1$" -Force -AsPlainText

$newSelfsignedCertificateExGithubUrl = 'https://raw.githubusercontent.com/Azure/azure-libraries-for-net/master/Samples/Asset/New-SelfSignedCertificateEx.ps1'

$newSelfSignedCertificateExFileName  = 'New-SelfSignedCertificateEx.ps1'
$downloadsFolder                     = $("$env:USERPROFILE\Downloads")
$newSelfSignedCertificateExFullPath  = Join-Path -Path $dscConfigFullPath -ChildPath $newSelfSignedCertificateExFileName

$dscSelfSignedCertificateName        = 'dscSelfSignedCertificate'
$dscSelfSignedCerCertificateName     = $dscSelfSignedCertificateName,'cer' -join '.'
$dscSelfSignedPfxCertificateName     = $dscSelfSignedCertificateName,'pfx' -join '.'

#endregion

#region create path for DSC outputs
If(!(Test-Path -Path $dscConfigFullPath)){
        try{
            New-Item -Path $dscConfigFullPath -ItemType Directory
        }
        catch{

        }
    }
#endregion

# set the location to the path where the DSC configuration is stored
Set-Location -Path $dscConfigPath
#endregion

#region - run once - DSC prerequisites
#region DSC - download prereq function
#Start-Process 'https://github.com/Azure/azure-libraries-for-net/blob/master/Samples/Asset/New-SelfSignedCertificateEx.ps1'
Invoke-WebRequest -Uri $newSelfsignedCertificateExGithubUrl -OutFile $newSelfSignedCertificateExFullPath
#endregion

#region DSC - Install Missing modules
# if the modules are not installed then
# the execution of 
#
# . .\ConfigureNode.ps1 
#
# throws errors

Install-Module -Name 'PSDscResources' -RequiredVersion 2.12.0.0 -Force -AllowClobber
Install-Module -Name 'ComputerManagementDsc' -RequiredVersion 9.1.0 -Force -AllowClobber
Install-Module -Name 'NetworkingDsc' -RequiredVersion 9.0.0 -Force -AllowClobber

#Get-Module -ListAvailable -Name 'NetworkingDsc'
#Get-Module -ListAvailable -Name 'ComputerManagementDsc'

#Get-Module -Name NetworkingDsc -ListAvailable        #not available
#Get-Module -Name ComputerManagementDsc -ListAvailable #v1.1
#endregion

#region DSC - create self signed certificates
$dscSelfSignedCerCertificateFullPath = Join-Path -Path $dscConfigFullPath -ChildPath $dscSelfSignedCerCertificateName
$dscSelfSignedPfxCertificateFullPath = Join-Path -Path $dscConfigFullPath -ChildPath $dscSelfSignedPfxCertificateName

. $newSelfSignedCertificateExFullPath

$selfSignedCertificate = @{
 Subject               = "CN=${ENV:ComputerName}"
 EKU                   = 'Document Encryption'
 KeyUsage              = 'KeyEncipherment, DataEncipherment'
 SAN                   = ${ENV:ComputerName}
 FriendlyName          = 'DSC Credential Encryption certificate'
 Exportable            = $true
 StoreLocation         = 'LocalMachine'
 KeyLength             = 2048
 ProviderName          = 'Microsoft Enhanced Cryptographic Provider v1.0'
 AlgorithmName         = 'RSA'
 SignatureAlgorithm    = 'SHA256'
}

#$selfSignedCertificate
#$selfSignedCertificate.FriendlyName

New-SelfsignedCertificateEx @selfSignedCertificate

Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-Certificate -Type cer -FilePath $dscSelfSignedCerCertificateFullPath -Force

# 3. export certificate (with Private key) to C:\DscPrivateKey.pfx

#Get-ChildItem -Path Cert:\LocalMachine\My\ | where{$_.Thumbprint -eq "4eeee9dca7dd5ccf70e47e46ac1128ddddbbb321"} | Export-PfxCertificate -FilePath "$env:USERPROFILE\Documents\dscSelfSignedCertificate\mypfx.pf" -Password $mypwd
Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-PfxCertificate -FilePath $dscSelfSignedPfxCertificateFullPath -Password $mypwd

#Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password $mypwd
#Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\Root -Password $mypwd

# now modify the ConfigData.psd1
# * update the CertificateFile location if needed
# * update the Thumbprint
(Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)}).Thumbprint | clip
psedit $dscConfigDataPath
#endregion
#endregion

#region - run once - LCM - configure certificate thumbprint
# Import the configuration data
#$ConfigData = .\ConfigData.psd1
$ConfigData = Import-PowerShellDataFile -Path $dscConfigDataPath
#$ConfigData.AllNodes

. .\ConfigureLCM.ps1

# Generate the MOF file for LCM configuration
ConfigureLCM -ConfigurationData $ConfigData -OutputPath $(Join-Path -Path $dscConfigFullPath -ChildPath 'LCM')

# Apply LCM configuration
Set-DscLocalConfigurationManager -Path $(Join-Path -Path $dscConfigFullPath -ChildPath 'LCM') -Verbose

# check LCM configuration
# for the CIM sessions to work the WIMrm should be configured first
Get-DscLocalConfigurationManager -CimSession localhost
#endregion

#region - run anytime - Import Configuration Data
$ConfigData = Import-PowerShellDataFile -Path $dscConfigDataPath
#$ConfigData.AllNodes
#psedit $dscConfigDataPath

. .\ConfigureNode.ps1

# Generate the MOF files and apply the configuration
# Credentials are used within the configuration file - hence SelfSigned certificate is needed as there is no Active Directory Certification Services
NodeInitialConfig -ConfigurationData $ConfigData -AdminCredential $AdminCredential -OutputPath $dscConfigFullPath -Verbose

Start-DscConfiguration -Path $dscConfigFullPath -Wait -Verbose -Force
#endregion
#endregion
```

At this point your management VM should have:

* name specified in psd1 file
* disabled IPv6 address
* defined trusted hosts
* WinRM service running

You are also ready for further DSC configuration to setup the on-premises Active Directory.

## Summary

Last update: 2024.06.25
