---
layout: post
title: "Windows 10 management VM - DSC"
permalink: "/windows-mgmt-vm-DSC/"
subtitle: "Initial Desired State Configuration setup of w10 mgmt VM"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft ,DSC]
categories: [HomeLab ,Microsoft ,DSC]
---
This Windows 10 VM is used as a starting point, acting as management node for the MS landscape. Only initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

* there is no Active Directory yet
* AD will be configured by making use of DSC

Goals:

* GUI based administration ise conducted here
* Once the domain is provisioned, w10mgmt becomes domain joned
* WinRM is ENABLED
* Initial Configuration conducted by Desired State Configuration

ToDo:

* Separate the DSC configuration
* Focus only on the tooling itself and it's installation
* OS   : Find the registry keys which configures the Edge
* WinRM: Configure the trusted host section with DSC

## 0. Links

* 

## 1. Assumptions

* Your network device of choice consists of a static reservation for the DHCP address, until you decide to go with the static IP - for the w10mgmt VM
* DNS on the w10mgmt node can resolve the public internet addresses

## 2. Howto

Run following code in elevated ISE session.

It creates [000_w10mgmt_initialConfig.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig_demo.ps1) in $env:USERPROFILE\Documents directory.

```powershell
#run ISE as Administartor
$dscCodeRepoUrl                         = 'https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration'
$dsc_000_w10mgmt_InitialConfig_FileName = '000_w10mgmt_initialConfig_demo.ps1'
$w10mgmt_initalConfig_demo_ps1_url      = $dscCodeRepoUrl,$dsc_000_w10mgmt_InitialConfig_FileName -join '/'

$outFile = Join-Path -Path $env:USERPROFILE\Documents -ChildPath $dsc_000_w10mgmt_InitialConfig_FileName
Invoke-WebRequest -Uri $w10mgmt_initalConfig_demo_ps1_url -OutFile $outFile

psedit $outFile
```

Run the whole 000_w10mgmt_initialConfig_demo.ps1 script in one go or code within regions:

### 2.1. Initialize variables

It initialize all variables for succesfull code execution.

### 2.2. Download the powershell functions and configuration

It downloads configurations from github and store in $env:SYSTEMDRIVE\dscConfig\_w10mgmt\000_w10mgmt_initialConfig:

* [ConfigData.psd1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig/ConfigData.psd1)
* [ConfigureLCM.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig/ConfigureLCM.ps1)
* [ConfigureNode.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig/ConfigureNode.ps1)

```powershell
# Files should be sotred in the $dscConfigDirectory
$env:SYSTEMDRIVE\dscConfig\_w10mgmt\000_w10mgmt_initialConfig\ConfigData.psd1
$env:SYSTEMDRIVE\dscConfig\_w10mgmt\000_w10mgmt_initialConfig\ConfigureLCM.ps1
$env:SYSTEMDRIVE\dscConfig\_w10mgmt\000_w10mgmt_initialConfig\ConfigureNode.ps1
```

### 2.3. DSC - Install missing modules

It installs missing modules for the DSC succesfull execution.

### 2.4. DSC - Self signed certificate preparation

It prepares self signed certificate for securing the credentials.

### 2.5. DSC - Certificate thumbprint update - ConfigData.psd1

It opens the ConfigData.psd1 file to update the Thumbprint of the certificate

### 2.6. LCM - Ammend certificate thumbprint

It configures the LCM with the certificate thumbprint.

### 2.7. DSC - Start Configuration

It starts the actual configuration of the node.

## 3. Content of 000_w10mgmt_initialConfig_demo.ps1

The [000_w10mgmt_initialConfig_demo.ps1](https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration/000_w10mgmt_initialConfig_demo.ps1) has the following content.

```powershell
#region - run once - prerequisites for the DSC to work properly
#region w10mgmt - initial checks
update-help
Get-ExecutionPolicy
Get-Service -Name WinRM #stopped
Test-WSMan -ComputerName localhost #can not connect 
#Get-Item WSMan:\localhost\Client\TrustedHosts #winRM is not running hence error during execution
#endregion

#region WinRM configuration
Set-NetConnectionProfile -NetworkCategory Private
Enable-PSRemoting
Get-Item WSMan:\localhost\Client\TrustedHosts #empty
#endregion
#endregion

#region - initialize variables
#region - initialize variables - DSC structure
$dscCodeRepoUrl                            = 'https://raw.githubusercontent.com/makeitcloudy/AutomatedLab/feature/007_DesiredStateConfiguration'
$dsc_000_w10mgmt_InitialConfig_FolderName  = '000_w10mgmt_initialConfig'
$dsc_000_w10mgmt_InitialConfig_FileName    = '000_w10mgmt_initialConfig_demo.ps1'

$dscCodeRepo_000_w10mgmt_initialConfig_url = $dscCodeRepoUrl,$dsc_000_w10mgmt_InitialConfig_FolderName -join '/'

$downloadsFolder                           = $("$env:USERPROFILE\Downloads")
$certificate_FolderName                    = '__certificate'

$dscSelfSignedCertificateName              = 'dscSelfSignedCertificate'
$dscSelfSignedCerCertificateName           = $dscSelfSignedCertificateName,'cer' -join '.'
$dscSelfSignedPfxCertificateName           = $dscSelfSignedCertificateName,'pfx' -join '.'

$newSelfSignedCertificateExFileName        = 'New-SelfSignedCertificateEx.ps1'
$newSelfsignedCertificateExGithubUrl       = 'https://raw.githubusercontent.com/Azure/azure-libraries-for-net/master/Samples/Asset',$newSelfSignedCertificateExFileName -join '/'

$selfSignedCertificate                     = @{
    Subject                                = "CN=${ENV:ComputerName}"
    EKU                                    = 'Document Encryption'
    KeyUsage                               = 'KeyEncipherment, DataEncipherment'
    SAN                                    = ${ENV:ComputerName}
    FriendlyName                           = 'DSC Credential Encryption certificate'
    Exportable                             = $true
    StoreLocation                          = 'LocalMachine'
    KeyLength                              = 2048
    ProviderName                           = 'Microsoft Enhanced Cryptographic Provider v1.0'
    AlgorithmName                          = 'RSA'
    SignatureAlgorithm                     = 'SHA256'
}

$dscConfig_FolderName                      = 'dscConfig'
$dscOutput_FolderName                      = '__output'
$lcm_FolderName                            = 'LCM'

$nodeName                                  = '_w10mgmt'

$configData_psd1_fileName                  = 'ConfigData.psd1'
$configureLCM_ps1_fileName                 = 'ConfigureLCM.ps1'
$configureNode_ps1_fileName                = 'ConfigureNode.ps1'

$configData_psd1_url                       = $dscCodeRepo_000_w10mgmt_initialConfig_url,$configData_psd1_fileName -join '/'
$configureLCM_ps1_url                      = $dscCodeRepo_000_w10mgmt_initialConfig_url,$configureLCM_ps1_fileName -join '/'
$configureNode_ps1_url                     = $dscCodeRepo_000_w10mgmt_initialConfig_url,$configureNode_ps1_fileName -join '/'

$dscConfigDirectoryPath                    = Join-Path -Path "$env:SYSTEMDRIVE" -childPath $dscConfig_FolderName
$dscConfigCertificateDirectoryPath         = Join-Path -Path $dscConfigDirectoryPath -ChildPath $certificate_FolderName
$dscConfigOutputDirectoryPath              = Join-Path -Path $dscConfigDirectoryPath -ChildPath $dscOutput_FolderName
$dscConfigNodeDirectoryPath                = Join-Path -Path $dscConfigDirectoryPath -ChildPath $nodeName
$dscConfig_000_w10mgmt_InitialConfig_Path =  Join-Path -Path $dscConfigNodeDirectoryPath -ChildPath $dsc_000_w10mgmt_InitialConfig_FolderName

$configData_psd1_FullPath                  = Join-Path -Path $dscConfig_000_w10mgmt_InitialConfig_Path -ChildPath $configData_psd1_fileName
$configureLCM_ps1_FullPath                 = Join-Path -Path $dscConfig_000_w10mgmt_InitialConfig_Path -ChildPath $configureLCM_ps1_fileName 
$configureNode_ps1_FullPath                = Join-Path -Path $dscConfig_000_w10mgmt_InitialConfig_Path -ChildPath $configureNode_ps1_fileName

$newSelfSignedCertificateExFullPath        = Join-Path -Path $dscConfigCertificateDirectoryPath -ChildPath $newSelfSignedCertificateExFileName
$dscSelfSignedCerCertificateFullPath       = Join-Path -Path $dscConfigCertificateDirectoryPath -ChildPath $dscSelfSignedCerCertificateName
$dscSelfSignedPfxCertificateFullPath       = Join-Path -Path $dscConfigCertificateDirectoryPath -ChildPath $dscSelfSignedPfxCertificateName

$dscConfigLCMDirectoryPath                 = Join-Path -Path $dscConfigOutputDirectoryPath -ChildPath $lcm_FolderName
#endregion

#region - initialize variables - credentials
# local administartor on the localhost
$localNodeAdminUsername                    = "labuser"
$localNodeAdminPassword                    = ConvertTo-SecureString "Password1$" -AsPlainText -Force
$localNodeAdminCredential                  = New-Object System.Management.Automation.PSCredential ($localNodeAdminUsername, $localNodeAdminPassword)

# creds for PFX self signed cert
$mypwd                                     = ConvertTo-SecureString -String "Password1$" -Force -AsPlainText
#endregion

#region - initialize variables - create folder structure
# create folder to store the DSC configuration
# TODO: change the folder creation to Desired State Configuration

If(!(Test-Path -Path $dscConfigDirectoryPath)){
        try{
            New-Item -Path $dscConfigDirectoryPath -ItemType Directory -Force -Verbose
        }
        catch{
            Write-Output "skipping"
        }
    }
else {
Write-Output "skipping"
}
If(!(Test-Path -Path $dscConfigNodeDirectoryPath)){
        try{
            New-Item -Path $dscConfigNodeDirectoryPath -ItemType Directory -Force -Verbose
        }
        catch{
            Write-Output "skipping"
        }
    }
else {
Write-Output "skipping"
}
If(!(Test-Path -Path $dscConfigCertificateDirectoryPath)){
        try{
            New-Item -Path $dscConfigCertificateDirectoryPath -ItemType Directory -Force -Verbose
        }
        catch{
            Write-Output "skipping"
        }
    }
else {
Write-Output "skipping"
}
If(!(Test-Path -Path $dscConfig_000_w10mgmt_InitialConfig_Path)){
        try{
            New-Item -Path $dscConfig_000_w10mgmt_InitialConfig_Path -ItemType Directory -Force -Verbose
        }
        catch{
            Write-Output "skipping"
        }
    }
else {
    Write-Output "skipping"
}
#endregion

# set the location to the path where the DSC configuration is stored
Set-Location -Path $dscConfigDirectoryPath
#endregion

#region - download the powershell functions and configuration
# download the helper functions and DSC configurations

Invoke-WebRequest -Uri $newSelfsignedCertificateExGithubUrl -OutFile $newSelfSignedCertificateExFullPath

Invoke-WebRequest -Uri $configData_psd1_url -OutFile $configData_psd1_FullPath
Invoke-WebRequest -Uri $configureLCM_ps1_url -OutFile $configureLCM_ps1_FullPath
Invoke-WebRequest -Uri $configureNode_ps1_url -OutFile $configureNode_ps1_FullPath

#endregion

Test-Path -Path $newSelfSignedCertificateExFullPath
. $newSelfSignedCertificateExFullPath
$env:COMPUTERNAME

#region DSC - Install Missing modules
# double check if this is a best practice
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

#region DSC - Self signed certificate preparation
New-SelfsignedCertificateEx @selfSignedCertificate

Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-Certificate -Type cer -FilePath $dscSelfSignedCerCertificateFullPath -Force
#export certificate (with Private key) to C:\DscPrivateKey.pfx
#Get-ChildItem -Path Cert:\LocalMachine\My\ | where{$_.Thumbprint -eq "4eeee9dca7dd5ccf70e47e46ac1128ddddbbb321"} | Export-PfxCertificate -FilePath "$env:USERPROFILE\Documents\dscSelfSignedCertificate\mypfx.pf" -Password $mypwd
Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-PfxCertificate -FilePath $dscSelfSignedPfxCertificateFullPath -Password $mypwd

#Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password $mypwd
#Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\Root -Password $mypwd
#endregion

#region DSC - Configuration data certificate thumbprint update
# now modify the ConfigData.psd1
# * update the CertificateFile location if needed
# * update the Thumbprint
(Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)}).Thumbprint | clip
psedit $configData_psd1_FullPath
#endregion



#region - run once - LCM - configure certificate thumbprint
# Import the configuration data
#$ConfigData = .\ConfigData.psd1
$ConfigData = Import-PowerShellDataFile -Path $configData_psd1_FullPath
#$ConfigData.AllNodes

#psedit $configureLCM_ps1_FullPath
#. .\ConfigureLCM.ps1
. $configureLCM_ps1_FullPath

# Generate the MOF file for LCM configuration
ConfigureLCM -ConfigurationData $ConfigData -OutputPath $dscConfigLCMDirectoryPath

# Apply LCM configuration
Set-DscLocalConfigurationManager -Path $dscConfigLCMDirectoryPath -Verbose

# check LCM configuration
# for the CIM sessions to work the WIMrm should be configured first
Get-DscLocalConfigurationManager -CimSession localhost
#endregion



#region - run anytime - Import Configuration Data - run DsC NodeInitialConfig
$ConfigData = Import-PowerShellDataFile -Path $configData_psd1_FullPath
#$ConfigData.AllNodes
#psedit $dscConfigDataPath

#psedit $configureNode_ps1_FullPath
#. .\ConfigureNode.ps1
. $configureNode_ps1_FullPath

# Generate the MOF files and apply the configuration
# Credentials are used within the configuration file - hence SelfSigned certificate is needed as there is no Active Directory Certification Services
NodeInitialConfig -ConfigurationData $ConfigData -AdminCredential $localNodeAdminCredential -OutputPath $dscConfigOutputDirectoryPath -Verbose

Start-DscConfiguration -Path $dscConfigOutputDirectoryPath -Wait -Verbose -Force
#endregion
```

At this point your management VM should have:

* name specified in psd1 file
* disabled IPv6 address
* defined trusted hosts
* WinRM service running

You are also ready for further DSC configuration to setup the on-premises Active Directory.

## Summary

Last update: 2024.07.01
