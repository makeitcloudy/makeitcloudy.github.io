---
layout: post
title: "Windows management VM - DSC"
permalink: "/windows-mgmt-vm-DSC/"
subtitle: "Initial Desired State Configuration setup management VM"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab ,Microsoft ,DSC]
categories: [HomeLab ,Microsoft ,DSC]
---
This Windows based VM is used as a starting point, acting as management node for the MS landscape. Only initial DSC configuration is held here, once the Active Directory domain is in place, the whole DSC work is aranged on dedicated authoring VM.

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
* PowerShell script execution policy is configured properly to run scripts - OK
* winRM service is running on the node - OK
* [AutomatedLab](https://github.com/makeitcloudy/AutomatedLab/tree/main) PowerShell Module Installed (otherwise it won't install the DSC modules properly)

## 2. Howto

### 2.1 Run an elevated powershell ISE instance

```powershell
# run PowerShell session
Start-Process PowerShell_ISE -Verb RunAs
```

### 2.2 Run following code in elevated ISE session.

It creates [InitialConfig_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/InitialConfig_setup.ps1) in $env:USERPROFILE\Documents directory.

Run the code below.

```powershell
#run in elevated powershell session
#region - initialize variables, downlad prereqs
$dsc_CodeRepoUrl               = 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration'
$dsc_InitialConfigFileName     = 'InitialConfig_setup.ps1'
$dsc_initalConfig_demo_ps1_url = $dsc_CodeRepoUrl,$dsc_InitialConfigFileName -join '/'

$outFile = Join-Path -Path $env:USERPROFILE\Documents -ChildPath $dsc_InitialConfigFileName
Invoke-WebRequest -Uri $dsc_initalConfig_demo_ps1_url -OutFile $outFile -Verbose

#psedit $outFile
#endregion

$NodeName = 'testnode' #FIXME: It equals to the computername (w10mgmt in this case)

#region - run it 
. $outFile
Set-InitialConfiguration -NodeName $NodeName -Option WorkGroup -Verbose
# The -UpdatePowerShellHelp Parameter
#Set-InitialConfiguration -NodeName $NodeName -Option WorkGroup -UpdatePowerShellHelp  -Verbose

```

* It initialize all variables for succesfull code execution.
* It downloads the powershell functions and configuration

It downloads configurations from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup:

* [ConfigData.psd1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1)
* [ConfigureLCM.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureLCM.ps1)
* [ConfigureNode.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureNode.ps1)

```powershell
# Files should be sotred in the $dscConfigDirectory
$env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup\ConfigData.psd1
$env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup\ConfigureLCM.ps1
$env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup\ConfigureNode.ps1
```

* It installs missing modules for the DSC succesfull execution.
* It prepares self signed certificate for securing the credentials.
* It takes care about the self signed certificate thumbprint and passes it as a paramter into the LCM configuration
* It configures the LCM
* It starts the actual configuration of the node.

## 3. Content of InitialConfig_setup.ps1

The [InitialConfig_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/InitialConfig_setup.ps1) has the following content.

```powershell
function Set-InitialConfiguration {

    <#
    .SYNOPSIS
    .DESCRIPTION
    .PARAMETER NodeName
    .PARAMETER Option
    .EXAMPLE
    .LINK
    #>
        
    [CmdletBinding()]
    Param
    (
        [Parameter(Mandatory=$true,Position=0,ValueFromPipelineByPropertyName=$true)]
        [ValidateNotNullOrEmpty()]
        [String]$NodeName,

        [Parameter(Mandatory=$true,Position=1,ValueFromPipelineByPropertyName=$true)]
        [ValidateNotNullOrEmpty()][ValidateSet("WorkGroup", "Domain")]
        [string]$Option,

        [Parameter(Mandatory=$false,Position=2,ValueFromPipelineByPropertyName=$false)]
        [switch]$UpdatePowerShellHelp
    )

    BEGIN
    {
        $WarningPreference = "Continue"
        $VerbosePreference = "Continue"
        $InformationPreference = "Continue"
        Write-Verbose "$env:COMPUTERNAME - $($MyInvocation.MyCommand) - Example"
        $startDate = Get-Date

        switch ($Option) {
            "Workgroup" {
                $workgroup = $true
                $domain = $false
                Write-Output "ParameterOne: $Option, Workgroup is enabled."
            }
            "Domain" {
                $workgroup = $false
                $domain = $true
                Write-Output "ParameterOne: $Option, Domain is enabled."
            }
        }

        #region 1. DSC Initialize variables
        #region Initialize Variables - Missing Modules
        $modules = @{
            'PSDscResources'                        = '2.12.0.0'
            #'ActiveDirectoryDsc'                    = '6.4.0'
            'ComputerManagementDsc'                 = '9.1.0'
            'NetworkingDsc'                         = '9.0.0'
        }
        #endregion

        #region - initialize variables - credentials
        # creds for PFX self signed cert
        $selfSignedCertificatePrivateKeyPasswordSecureString = ConvertTo-SecureString -String "Password1$" -Force -AsPlainText
        #endregion

        #region Initialize Variables - Folder structure
        $dsc_FolderName                            = 'dsc'             #C:\dsc\

        $config_FolderName                         = 'config'          #C:\dsc\config\ - # it stores the DSC configuration
        $localhost_FolderName                      = 'localhost'       #C:\dsc\config\localhost\ - # it stores the configurations run locally on the localhost (Node)
    
        $certificate_FolderName                    = 'certificate'     #C:\dsc\certificate\ - # it stores self signed certificate used to secure DSC credentials
        $function_FolderName                       = 'function'        #C:\dsc\function\
        $module_FolderName                         = 'module'          #C:\dsc\module\ - # it stores the Module which contains various functions
    
        $output_FolderName                         = '_output'         #C:\dsc\_output\ - # it stores the results of the DSC compilation
        $LCM_FolderName                            = 'LCM'             #C:\dsc\_output\LCM\
    
        $InitialSetup_FolderName                   = 'InitialSetup'    #C:\dsc\config\localhost\InitialSetup\ - # it contains the configurations of ActiveDirectory
    
        $dscCodeRepoUrl                            = 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration'
        $dsc_000_InitialConfig_FolderName          = '000_initialConfig'
        #$dsc_000_InitialConfig_FileName            = '000_initialConfig_demo.ps1'

        $dscCodeRepo_000_initialConfig_url         = $dscCodeRepoUrl,$dsc_000_InitialConfig_FolderName -join '/'

        $configData_psd1_fileName                  = 'ConfigData.psd1'
        $configureLCM_ps1_fileName                 = 'ConfigureLCM.ps1'
        $configureNode_ps1_fileName                = 'ConfigureNode.ps1'

        $configData_psd1_url                       = $dscCodeRepo_000_initialConfig_url,$configData_psd1_fileName -join '/'
        $configureLCM_ps1_url                      = $dscCodeRepo_000_initialConfig_url,$configureLCM_ps1_fileName -join '/'
        $configureNode_ps1_url                     = $dscCodeRepo_000_initialConfig_url,$configureNode_ps1_fileName -join '/'

    
        #C:\dsc\
        $dsc_DirectoryPath                         = Join-Path -Path "$env:SYSTEMDRIVE" -childPath $dsc_FolderName
    
        #C:\dsc\config\
        $dscConfig_DirectoryPath                   = Join-Path -Path $dsc_DirectoryPath -childPath $config_FolderName
        #C:\dsc\config\localhost\
        $dscConfigLocahost_DirectoryPath           = Join-Path -Path $dscConfig_DirectoryPath -ChildPath $localhost_FolderName
        #C:\dsc\config\localhost\InitialSetup\
        $dscConfigLocalhostInitialSetup_DirectoryPath  = Join-Path -Path $dscConfigLocahost_DirectoryPath -ChildPath $InitialSetup_FolderName
    
        #C:\dsc\certificate\
        $dscCertificate_DirectoryPath              = Join-Path -Path $dsc_DirectoryPath -ChildPath $certificate_FolderName
    
        #C:\dsc\function\
        $dscFunction_DirectoryPath                 = Join-Path -Path $dsc_DirectoryPath -ChildPath $function_FolderName
    
        #C:\dsc\module\
        $dscModule_DirectoryPath                   = Join-Path -Path $dsc_DirectoryPath -ChildPath $module_FolderName
    
        #C:\dsc\_output\
        $dscOutput_DirectoryPath                   = Join-Path -Path $dsc_DirectoryPath -ChildPath $output_FolderName
        #C:\dsc\_output\InitialSetup
        $dscOutputInitialSetup_DirectoryPath       = Join-Path -Path $dscOutput_DirectoryPath -ChildPath $InitialSetup_FolderName
        #C:\dsc\_output\LCM
        $dscOutputLCM_DirectoryPath               = Join-Path -Path $dscOutput_DirectoryPath -ChildPath $LCM_FolderName

        $configData_psd1_FullPath                  = Join-Path -Path $dscConfigLocalhostInitialSetup_DirectoryPath -ChildPath $configData_psd1_fileName
        $configureLCM_ps1_FullPath                 = Join-Path -Path $dscConfigLocalhostInitialSetup_DirectoryPath -ChildPath $configureLCM_ps1_fileName 
        $configureNode_ps1_FullPath                = Join-Path -Path $dscConfigLocalhostInitialSetup_DirectoryPath -ChildPath $configureNode_ps1_fileName
        #endregion

        #region Initialize Variables - Function - C:\dsc\function\New-SelfSignedCertificateEx.ps1
        # it contains the Function to prepare the self signed certificate
        $newSelfSignedCertificateEx_FileName       = 'New-SelfSignedCertificateEx.ps1'
        $newSelfsignedCertificateEx_GithubUrl      = 'https://raw.githubusercontent.com/Azure/azure-libraries-for-net/master/Samples/Asset',$newSelfSignedCertificateEx_FileName -join '/'

        $dscFunction_NewSelfSignedCertificateEx_FullPath = Join-Path -Path $dscFunction_DirectoryPath -ChildPath $newSelfSignedCertificateEx_FileName

        $selfSignedCertificateParams               = @{
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

        $selfSignedCertificatePrivateKeyPassword              = 'Password1$'
        $selfSignedCertificatePrivateKeyPasswordSecureString  = ConvertTo-SecureString -String $selfSignedCertificatePrivateKeyPassword -Force -AsPlainText
        #endregion

        #region Initialize Variables - Self Signed Certificate
        $dscSelfSignedCertificate_FileName         = 'dscSelfSignedCertificate'
        $dscSelfSignedCerCertificate_FileName      = $dscSelfSignedCertificate_FileName,'cer' -join '.'
        $dscSelfSignedPfxCertificate_FileName      = $dscSelfSignedCertificate_FileName,'pfx' -join '.'

        $dscSelfSignedCerCertificate_FullPath      = Join-Path -Path $dscCertificate_DirectoryPath -ChildPath $dscSelfSignedCerCertificate_FileName
        $dscSelfSignedPfxCertificate_FullPath      = Join-Path -Path $dscCertificate_DirectoryPath -ChildPath $dscSelfSignedPfxCertificate_FileName
        #endregion
        #endregion

    }

    PROCESS
    {
        if ($PSBoundParameters.ContainsKey('UpdatePowerShellHelp')) {
            Write-Information "Update PowerShell Help"
            try
            {
                Update-Help | Out-Null
                #Get-ExecutionPolicy
                #Test-WSMan -ComputerName localhost #can not connect 
                #Get-Item WSMan:\localhost\Client\TrustedHosts #winRM is not running hence error during execution
                #endregion            
            }
            catch 
            {
    
            }

        } else {
            Write-Information "Skipping the PowerShell Help Update Process"
        }

        try
        {
            #WinRM configuration
            Write-Information "Enable WinRM"
            if(!(Get-service -Name "WinRm").Status -eq 'Running'){
                Get-Service -Name WinRM | Start-Service | Out-Null
            }
            
        }
        catch
        {

        }

        try
        {
            Write-Information "Set NetConnectionProfile to Private"
            Set-NetConnectionProfile -NetworkCategory Private | Out-Null
            #Enable-PSRemoting
            #Get-Item WSMan:\localhost\Client\TrustedHosts #empty
            #endregion
        }
        catch
        {

        }

        try
        {
            #region 2.1. Create Directory Structure
            $arrayFolderStructure = @(
                $dsc_DirectoryPath,
                $dscConfig_DirectoryPath,
                $dscConfigLocahost_DirectoryPath,
                $dscConfigLocalhostInitialSetup_DirectoryPath,
                $dscCertificate_DirectoryPath,
                $dscFunction_DirectoryPath,
                $dscModule_DirectoryPath,
                $dscOutput_DirectoryPath,
                $dscOutputInitialSetup_DirectoryPath,
                $dscOutputLCM_DirectoryPath
            )

            $arrayFolderStructure.ForEach({
                if(!(Test-Path -Path $_)){
                    try {
                        Write-Information "Create Directory: $_"
                        New-Item -Path $_ -ItemType Directory -Force | Out-Null
                    }
                    catch {
                        Write-Error "Something went wrong"
                    }
                }
                else {
                    Write-Warning "$_ - already exist"
                }
            })

            #endregion
        }
        catch {

        }

        try {
            #region 2.2. Download code from from Github
            # Function: SelfSigned Certificate
            #https://raw.githubusercontent.com/Azure/azure-libraries-for-net/master/Samples/Asset/New-SelfSignedCertificateEx.ps1
            Write-Information "Downloading: $newSelfsignedCertificateEx_GithubUrl"
            Invoke-WebRequest -Uri $newSelfsignedCertificateEx_GithubUrl -OutFile $dscFunction_NewSelfSignedCertificateEx_FullPath -Verbose
            #endregion
        }
        
        catch
        {

        }

        try
        {
            # Function: DSC Configuration
            Write-Information "Downloading: $configData_psd1_url"
            Invoke-WebRequest -Uri $configData_psd1_url -OutFile $configData_psd1_FullPath -Verbose
            Write-Information "Downloading: $configureLCM_ps1_url"
            Invoke-WebRequest -Uri $configureLCM_ps1_url -OutFile $configureLCM_ps1_FullPath -Verbose
            Write-Information "Downloading: $configureNode_ps1_url"
            Invoke-WebRequest -Uri $configureNode_ps1_url -OutFile $configureNode_ps1_FullPath -Verbose
        #endregion
        }
        catch
        {

        }

        # set the location to the path where the DSC configuration is stored
        Write-Information "Change current directory: $dscConfig_DirectoryPath"
        Set-Location -Path $dscConfig_DirectoryPath
        #endregion

        try
        {
            #Test-Path -Path $newSelfSignedCertificateExFullPath
            #. $newSelfSignedCertificateExFullPath
            #$env:COMPUTERNAME
            Write-Information "Set Execution Policy: Bypass, Scope LocalMachine"
            Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force | Out-Null
            Write-Information "Install Nuget"
            Install-PackageProvider -Name Nuget -MinimumVersion 2.8.5.201 -Force | Out-Null

            #region DSC - Intall missing modules
            Write-Information "Install Missing DSC Modules"
            Install-Modules -modules $modules
            #endregion

            #region DSC - Install missing modules
            # double check if this is a best practice
            ## Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope LocalMachine -Force

            ## Install-PackageProvider -Name Nuget -MinimumVersion 2.8.5.201 -Force

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

            ## Install-Module -Name 'PSDscResources' -RequiredVersion 2.12.0.0 -Force -AllowClobber
            ## Install-Module -Name 'ComputerManagementDsc' -RequiredVersion 9.1.0 -Force -AllowClobber
            ## Install-Module -Name 'NetworkingDsc' -RequiredVersion 9.0.0 -Force -AllowClobber

            #Get-Module -ListAvailable -Name 'NetworkingDsc'
            #Get-Module -ListAvailable -Name 'ComputerManagementDsc'

            #Get-Module -Name NetworkingDsc -ListAvailable        #not available
            #Get-Module -Name ComputerManagementDsc -ListAvailable #v1.1
            #endregion
        }

        catch
        {

        }
        
        try
        {
             #region 2.3. Self Signed Certificate - Generate and Export to CER & PFX
            if (!(Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificateParams.FriendlyName)})){
            # Load the function into memory
                . $dscFunction_NewSelfSignedCertificateEx_FullPath
                try {
                    Write-Information "Create Self Signed Certificate"
                    New-SelfsignedCertificateEx @selfSignedCertificateParams | Out-Null
                }
                catch {

                }
            }
            else {
                Write-Output "Certificate already exist - Friendly Name: $($selfSignedCertificateParams.FriendlyName)"
            }

            # Exporting certificate to CER and PFX
            if(!(Test-Path -Path $dscSelfSignedCerCertificate_FullPath)){
                try {
                    Write-Information "Export Self Signed Certificate to file: $dscSelfSignedCerCertificate_FullPath"
                    Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificateParams.FriendlyName)} | Export-Certificate -Type cer -FilePath $dscSelfSignedCerCertificate_FullPath -Force | Out-Null
                }
                catch {

                }
            }
            else {
                Write-Output "Certificate CER File already exist - Path: $($dscSelfSignedCerCertificate_FullPath)"
            }

            if(!(Test-Path -Path $dscSelfSignedPfxCertificate_FullPath)){
                try {
                    Write-Information "Export Self Signed Certificate to file: $$dscSelfSignedPfxCertificate_FullPath"
                    Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificateParams.FriendlyName)} | Export-PfxCertificate -FilePath $dscSelfSignedPfxCertificate_FullPath -Password $selfSignedCertificatePrivateKeyPasswordSecureString | Out-Null
                }
                catch {

                }    
            }
            else {
                Write-Output "Certificate PFX File already exist - Path: $($dscSelfSignedPfxCertificate_FullPath)"
            }
            #endregion

            #region DSC - Self signed certificate preparation
            ##New-SelfsignedCertificateEx @selfSignedCertificate

            ##Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-Certificate -Type cer -FilePath $dscSelfSignedCerCertificateFullPath -Force
            #export certificate (with Private key) to C:\DscPrivateKey.pfx
            #Get-ChildItem -Path Cert:\LocalMachine\My\ | where{$_.Thumbprint -eq "4eeee9dca7dd5ccf70e47e46ac1128ddddbbb321"} | Export-PfxCertificate -FilePath "$env:USERPROFILE\Documents\dscSelfSignedCertificate\mypfx.pf" -Password $mypwd
            ##Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)} | Export-PfxCertificate -FilePath $dscSelfSignedPfxCertificateFullPath -Password $mypwd

            #Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password $mypwd
            #Import-PfxCertificate -FilePath "$env:SystemDrive\Temp\dscSelfSignedCertificate.pfx" -CertStoreLocation Cert:\LocalMachine\Root -Password $mypwd
            #endregion
        }
        catch
        {

        }
        
        try
        {
            #region DSC - Certificate thumbprint update - ConfigData.psd1
            # now modify the ConfigData.psd1
            # * update the CertificateFile location if needed
            # * update the Thumbprint
            Write-Information "Get Self Signed Certificate Thumbprint"
            $selfSignedCertificateThumbprint = (Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificateParams.FriendlyName)}).Thumbprint
            #(Get-ChildItem -Path Cert:\LocalMachine\My\ | Where-Object {$_.FriendlyName -eq $($selfSignedCertificate.FriendlyName)}).Thumbprint | clip
            #psedit $configData_psd1_FullPath
            #endregion
        }
        catch
        {

        }

        try
        {
            #region - run once - LCM - Ammend certificate thumbprint
            # Import the configuration data
            #$ConfigData = .\ConfigData.psd1
            Write-Information "Import DSC Configuration Data: $configData_psd1_FullPat"
            $ConfigData = Import-PowerShellDataFile -Path $configData_psd1_FullPath
            #$ConfigData.AllNodes

            #psedit $configureLCM_ps1_FullPath
            #. .\ConfigureLCM.ps1
            Write-Information "Load DSC Configuration Data into Memory"
            . $configureLCM_ps1_FullPath

            # Generate the MOF file for LCM configuration
            Write-Information "Prepare the meta.mof for the LCM"
            ConfigureLCM -CertificateThumbprint $selfSignedCertificateThumbprint -ConfigurationData $ConfigData -OutputPath $dscOutputLCM_DirectoryPath | Out-Null

            # Apply LCM configuration
            Write-Information "Apply LCM Configuration"
            Set-DscLocalConfigurationManager -Path $dscOutputLCM_DirectoryPath -Verbose

            # check LCM configuration
            # for the CIM sessions to work the WIMrm should be configured first
            Write-Information "Get LCM Configuration details "
            Get-DscLocalConfigurationManager -CimSession localhost
            #endregion
        }
        catch 
        {

        }

        try
        {

            # local administartor on the localhost
            $localNodeAdminUsername                    = "labuser"
            $localNodeAdminPassword                    = ConvertTo-SecureString "Password1$" -AsPlainText -Force
            $localNodeAdminCredential                  = New-Object System.Management.Automation.PSCredential ($localNodeAdminUsername, $localNodeAdminPassword)

            #region DSC - run anytime - Start Configuration
            $ConfigData = Import-PowerShellDataFile -Path $configData_psd1_FullPath
            #$ConfigData.AllNodes
            #psedit $dscConfigDataPath

            #psedit $configureNode_ps1_FullPath
            #. .\ConfigureNode.ps1
            . $configureNode_ps1_FullPath

            if($Workgroup){
                # Generate the MOF files and apply the configuration
                # Credentials are used within the configuration file - hence SelfSigned certificate is needed as there is no Active Directory Certification Services
                Write-Information "Start Node Initial Configuration - Option: Workgroup"
                NodeInitialConfigWorkgroup -ConfigurationData $ConfigData -NodeName $NodeName -AdminCredential $localNodeAdminCredential -OutputPath $dscOutputInitialSetup_DirectoryPath | Out-Null
            }
            if($domain){
                # Generate the MOF files and apply the configuration
                # Credentials are used within the configuration file - hence SelfSigned certificate is needed as there is no Active Directory Certification Services
                Write-Information "Start Node Initial Configuration - Option: Domain"
                NodeInitialConfigDomain -ConfigurationData $ConfigData -NodeName $NodeName -AdminCredential $localNodeAdminCredential -OutputPath $dscOutputInitialSetup_DirectoryPath | Out-Null
            }
            

            #Start-DscConfiguration -Path $dscConfigOutput_DirectoryPath -Wait -Verbose -Force
            Start-DscConfiguration -Path $dscOutputInitialSetup_DirectoryPath -Credential $localNodeAdminCredential -Wait -Verbose -Force
            #endregion
        }
        catch {

        }
    }

    END
    {
        $endDate = Get-Date
        Write-Verbose "$env:COMPUTERNAME - $($MyInvocation.MyCommand) - Time taken: $("{0:%d}d:{0:%h}h:{0:%m}m:{0:%s}s" -f ((New-TimeSpan -Start $startDate -End $endDate)))"
    }
}

```

At this point your management VM should have:

* new name, which equals to the $NodeName variable passed in paragraph 2.2, at the execution stage

```powershell
Set-InitialConfiguration -NodeName $NodeName -Option WorkGroup -Verbose
```

* disabled IPv6 address
* defined trusted hosts
* WinRM service running

You are also ready for further DSC configuration to setup the on-premises Active Directory.

## Summary

It was tested on: 

* Windows 10 (22H2 - 19045.4529)
* Server 2022 (21H2 - 20348.1547)

Last update: 2024.07.05