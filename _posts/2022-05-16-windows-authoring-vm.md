---
full-width: true
layout: post
title: "Windows DSC authoring VM"
permalink: "/windows-authoring-vm/"
subtitle: "Setup Desired State Configuration Authoring box"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft]
categories: [HomeLab, Microsoft]
---
This Windows 10 VM is used for authoring the Desired State Configuration configurations.

# Authoring Node - w10_authoring node

* Here the Desired state configuration is prepared and invoked.
* The IP address of this node should be included 

## 0. Prerequisites

* PowerShell 5.x
* PowerShell 7.x
* Visual Studio Code
* Git
* Desired State Configuration Resources

* extra disk: DSC Configurations, Helper functions, powershell modules

* initial configuration arranged by Desired State Configuration

## 1. Provisioning

At this point it is assumed:

1. There is SSH connectivity to the XCP-ng box
2. the xcp-ng scripts (used to provision vm's) are stored in /opt/scripts/

```bash
/opt/scripts/vm_create.sh --VmName 'w10_mgmt' --VCpu 4 --CoresPerSocket 2 --MemoryGB 8 --DiskGB 40 --ActivationExpiration 90 --TemplateName 'Windows 10 (64-bit)' --IsoName 'w10ent_21H2_updt_2302_unattended_noprompt.iso' --IsoSRName 'node1_nfs' --NetworkName 'NIC0 - .5.x' --Mac '5E:16:3e:5d:1f:05' --StorageName 'node1_ssd_sdb' --VmDescription 'mgmtBox'
```

## 2. Configuration

### 2.1

```powershell
# 1. update powershell help

# 3. download ADK
# 
# 
```

### 2.2

```powershell
#1. initialize disk - GPT | O: drive | NTFS
#2. run ISE elevated
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

## Summary

Last update: 2024.06.25
