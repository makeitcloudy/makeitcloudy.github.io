---
layout: post
title: "Windows management plane"
permalink: "/windows-mgmt-vm/"
subtitle: "Setup windows management box"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-windows.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft]
categories: [HomeLab ,Microsoft]
---
This Windows 10 VM is used as management node for the MS landscape. It is used as a starting point. Neither DSC authoring or OSD Image Factory are taking place here - those activities are performed on separate boxes... Only the initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

# MGMT Node - w10_mgmt node

Goals:

* GUI based administration ise conducted here
* Once the domain is provisioned, w10_mgmt becomes domain joned
* WinRM is ENABLED
* Initial Configuration conducted by Desired State Configuration

Todo:
* Separate the DSC configuration
* Focus only on the tooling itself and it's installation

## 0. Links

## 1. Prerequisites

* PowerShell 5.x
* PowerShell 7.x
* Management Box - RSAT tooling
* SQL Management Studio
* XenServer PowerShell module
* AutomatedXCP-ng module
* Visual Studio Code
* Git
* Filezilla

* extra disk: A:

* folder structure (this piece is prepared by Desired State Configuration)

```powershell
# A:\_repo
# A:\_repo\scripts_ps
# A:\_repo\scripts_shell
# A:\_repo\software
```

## 2. Provisioning

At this point it is assumed:

1. There is SSH connectivity to the XCP-ng box
2. the xcp-ng scripts (used to provision vm's) are stored in /opt/scripts/

```bash
/opt/scripts/vm_create.sh --VmName 'w10_mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_updt_2302_unattended_noprompt.iso' --IsoSRName 'node4_nfs' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:99:50' --StorageName 'node4_ssd_sdg' --VmDescription 'w10_mgmt_node'
```

## 3. Initial Configuration

 initial configuration is arranged by Desired State Configuration

```powershell
Start-Process PowerShell_ISE.exe -Verb RunAs

Get-ExecutionPolicy
# double check if this is a best practice
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
```
### 3.1 Extra disk

#### 3.1.1 Add disk

```bash
# run over SSH
/opt/scripts/vm_add_disk.sh --vmName "w10_mgmt" --storageName "node4_hdd_sdc_lsi" --diskName "w10_mgmt_dataDrive" --deviceId 4 --diskGB 20  --description "w10_mgmt_dataDrive"
```

#### 3.1.2 Initialize disk

```powershell
# here is the code for the disk initialization
```

### 3.2 Download Prerequisites

Download Citrix Hypervisor / XenTools 

#### 3.2.1 XenTools

Download XenTools

```powershell
# 1. Download VMTools (version 9.3.3)
# 2. TODO Create ISO (with the XS VMTools) from the directory - imgburn
```

#### 3.2.2 ImgBurn

It is used to prepare an ISO which contains XenTools

```powershell

```

### 3.3 RSAT Tools

Installation of RSAT tools

```powershell
# RSAT tools on the Desktop OS can NOT be installed by making use of DSC - it throws an error

#Get-WindowsCapability -Name RSAT* -Online | Select-Object -Property DisplayName, Name, State
#Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online

#Get-WindowsCapability -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0* -Online | Add-WindowsCapability -Online

Install-WindowsFeature RSAT-RDS-Licensing-Diagnosis-UI

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

[Install SQL Management Studio](https://qawithexperts.com/article/sql/download-and-install-sql-server-management-studio-step-by-st/441)
[Install SQL Server](https://qawithexperts.com/article/sql/download-and-install-sql-server-step-by-step-procedure/311)

```powershell
# https://www.sqlservercentral.com/articles/install-ssms-using-powershell-dsc

```

## 4. Tools

Installation of:
* Citrix Hypervisor SDK
* AutomatedXCPng - PowerShell module to orchestrate XCP-ng / XenServer

### 4.1 AutomatedXCPng

```powershell
# 5. Download https://github.com/makeitcloudy/AutomatedXCPng | Extract Zip file

# Copy AutomatedXCPng  module to C:\Program Files\WindowsPowerShell\Modules
# follow the guidelines: https://github.com/makeitcloudy/AutomatedXCPng

# XenPLModule folder should be copied into this location
# C:\Program Files\WindowsPowerShell\Modules
```

### 4.2 Citrix Hypervisor SDK

Download XenServer SDK

```powershell

# 1. Login to citrix.com
# 2. Switch to my account
# 3. Download XenServer 8.2.1 LTSR SDK - https://www.citrix.com/downloads/citrix-hypervisor/product-software/hypervisor-82-premium-edition-CU1.html | 
# 3. Unblock the downloaded ZIP file - if you do not unblock the zip file you will end up with an error about something going wrong with the .dll file once importing module, once unblocked everything is fine
# 4. Once unblocked - Extract Zip file

# From the extracted zip copy the XenServerPSModule from the CitrixHypervisor-SDK\XenServerPowerShell folder to 

# C:\Program Files\WindowsPowerShell\Modules directory or
# $env:UserProfile\Documents\WindowsPowerShell\Modules for per-user configuration or 
# $env:windir\system32\WindowsPowerShell\v1.0\Modules  for system-wide configuration

# run new powershell session elevated - so it reloads the copied modules
```

## 5. WinRM

ToDo: Configure the trusted host section with DSC

```powershell
#region run this on the client
Enable-PSRemoting
Set-NetConnectionProfile -NetworkCategory Private

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


Get-service -name winRM | Start-Service
get-Item WSMan:\localhost\Client\TrustedHosts

Set-Item wsman:\localhost\client\TrustedHosts -Value 10.2.134.210 -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 10.2.134.211 -Concatenate

$credential = Get-Credential

Enter-PSSession -ComputerName 10.2.134.210 -Credential $credential
Enter-PSSession -ComputerName 10.2.134.211 -Credential $credential
#endregion
```

## 6. DSC - Desired State Configuration

Preparation for the desired state configuration

```powershell
#1. Download to A:\_repo\scripts_ps
https://gist.github.com/makeitcloudy/05d5c4f48b2a143871c583609a150067
```

## 6.1 DSC - Initial Configuration

```powershell
# howto: https://amatijasec.wordpress.com/2017/12/12/how-to-use-powershell-dsc-to-deploy-active-directory-on-windows-server-2012-r2/

# 1. download the New-SelfSignedCertificateEx

https://github.com/Azure/azure-libraries-for-net/blob/master/Samples/Asset/New-SelfSignedCertificateEx.ps1

Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser

# ToDo: corret the path so it is pointing towards the Extra drive _repo folder

Set-Location -Path A:\_repo\scripts_ps
#Set-Location -Path .\dsc_prereq

. .\New-SelfSignedCertificateEx.ps1

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

$selfSignedCertificate
$selfSignedCertificate.FriendlyName

New-SelfsignedCertificateEx @selfSignedCertificate

Get-ChildItem
 
# 2. export certificate (with Public key only) to C:\DscPublicKey.cer

#https://medium.com/thesecmaster/how-to-export-a-certificate-from-powershell-a826cce955c5#48ab
Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-Certificate -Type cer -FilePath "$env:USERPROFILE\Documents\dscSelfSignedCertificate.cer" -Force

# 3. export certificate (with Private key) to C:\DscPrivateKey.pfx
$mypwd = ConvertTo-SecureString -String "Password1$" -Force -AsPlainText
#Get-ChildItem -Path Cert:\LocalMachine\My\ | where{$_.Thumbprint -eq "4eeee9dca7dd5ccf70e47e46ac1128ddddbbb321"} | Export-PfxCertificate -FilePath "$env:USERPROFILE\Documents\dscSelfSignedCertificate\mypfx.pf" -Password $mypwd
Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-PfxCertificate -FilePath "$env:USERPROFILE\Documents\dscSelfSignedCertificate.pfx" -Password $mypwd

# 4. On Target Nodes (DC01, DC02) import certificate (with Private key) into the Local Machine:
# Personal certificate store
# Trusted Root Certification Authorities certificate store

$winrm01 = New-PSSession -ComputerName '10.2.134.211'
$winrm02 = New-PSSession -ComputerName '10.2.134.212'

Invoke-Command -Session $winrm01 -ScriptBlock {New-Item -Path $env:SystemDrive\Temp -ItemType Directory}
Invoke-Command -Session $winrm02 -ScriptBlock {New-Item -Path $env:SystemDrive\Temp -ItemType Directory}

Copy-Item -Path "$env:USERPROFILE\Documents\dscSelfSignedCertificate.cer" -Destination "$env:SystemDrive\Temp" -ToSession $winrm01 -Verbose
Copy-Item -Path "$env:USERPROFILE\Documents\dscSelfSignedCertificate.pfx" -Destination "$env:SystemDrive\Temp" -ToSession $winrm01 -Verbose

Copy-Item -Path "$env:USERPROFILE\Documents\dscSelfSignedCertificate.cer" -Destination "$env:SystemDrive\Temp" -ToSession $winrm02 -Verbose
Copy-Item -Path "$env:USERPROFILE\Documents\dscSelfSignedCertificate.pfx" -Destination "$env:SystemDrive\Temp" -ToSession $winrm02 -Verbose

Invoke-Command -Session $winrm01 -ScriptBlock {Get-ChildItem -Path Cert:\LocalMachine\My}
Invoke-Command -Session $winrm01 -ScriptBlock {Get-ChildItem -Path Cert:\LocalMachine\Root}

Invoke-Command -Session $winrm02 -ScriptBlock {Get-ChildItem -Path Cert:\LocalMachine\My}
Invoke-Command -Session $winrm02 -ScriptBlock {Get-ChildItem -Path Cert:\LocalMachine\Root}

Invoke-Command -Session $winrm01 -ScriptBlock {Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password $using:mypwd}
Invoke-Command -Session $winrm01 -ScriptBlock {Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\Root -Password $using:mypwd}

Invoke-Command -Session $winrm02 -ScriptBlock {Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password $using:mypwd}
Invoke-Command -Session $winrm02 -ScriptBlock {Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\Root -Password $using:mypwd}
```

## 6.2 DSC

## 6.2.1 DSC - initial configuration

Initial Configuration which prepares the Folder Structure

```powershell
# save it as a dsc_config_InitialSetup.ps1
# A:\_repo
# A:\_repo\scripts_ps
# A:\_repo\scripts_shell
# A:\_repo\software

configuration InitialSetup {

    Import-DscResource -ModuleName PSDesiredStateConfiguration

          File Repo {
            Type = 'Directory'
            DestinationPath = 'A:\_repo'
            Ensure = "Present"
          }

          File RepoScripts {
            Type = 'Directory'
            DestinationPath = 'C:\_repo\scripts_ps'
            DependsOn = '[File]Repo'
          }

          File RepoScripts {
            Type = 'Directory'
            DestinationPath = 'C:\_repo\scripts_shell'
            DependsOn = '[File]Repo'
          }

          File RepoScripts {
            Type = 'Directory'
            DestinationPath = 'C:\_repo\software'
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
