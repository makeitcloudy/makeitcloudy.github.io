---
layout: post
title: "Windows management VM - DSC"
permalink: "/windows-DSC/"
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

**Goals**:

* GUI based administration ise conducted here
* Once the domain is provisioned, w10mgmt becomes domain joned
* WinRM is ENABLED
* Initial Configuration conducted by Desired State Configuration

**ToDo**:

* Separate the DSC configuration
* Focus only on the tooling itself and it's installation
* OS   : Find the registry keys which configures the Edge
* WinRM: Configure the trusted host section with DSC

## 0. Links

* .

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
#Start-Process PowerShell_ISE -Verb RunAs
#run in elevated powershell session
#region - initialize variables, downlad prereqs
$dsc_CodeRepoUrl               = 'https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode'
$dsc_InitialConfigFileName     = 'InitialConfigDsc.ps1'
$dsc_initalConfig_demo_ps1_url = $dsc_CodeRepoUrl,$dsc_InitialConfigFileName -join '/'

$outFile = Join-Path -Path $env:USERPROFILE\Documents -ChildPath $dsc_InitialConfigFileName
Invoke-WebRequest -Uri $dsc_initalConfig_demo_ps1_url -OutFile $outFile -Verbose

#psedit $outFile
#endregion

$NodeName = 'testnode' #FIXME: It equals to the computername (w10mgmt in this case)

#region - run it 
. $outFile
Set-InitialConfigurationDsc -NewComputerName $NodeName -Option WorkGroup -Verbose
# The -UpdatePowerShellHelp Parameter
#Set-InitialConfiguration -NewComputerName $NodeName -Option WorkGroup -UpdatePowerShellHelp  -Verbose

```

### 2.3. What does the code above do

* It initialize all variables for succesfull code execution
* It downloads the powershell functions and configuration
* It downloads [ConfigData.psd1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup
* It downloads [ConfigureLCM.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureLCM.ps1) from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup
* It downloads [ConfigureNode.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureNode.ps1) from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup
* It installs missing modules for the DSC succesfull execution
* It prepares self signed certificate for securing the credentials
* It takes care about the self signed certificate thumbprint and passes it as a paramter into the LCM configuration
* It configures the LCM
* It starts the actual configuration of the node

## 3. InitialConfig_setup.ps1 - essence

It has the following content [InitialConfig_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/InitialConfig_setup.ps1). Once run

```powershell
Set-InitialConfiguration -NewComputerName $NodeName -Option WorkGroup -Verbose
```

the target node should have:

* new name, which equals to the $NewComputerName variable
* disabled IPv6 address
* defined trusted hosts
* WinRM service running

Now the efforts with the Active Directory setup can be made.

## Summary

It was tested on: 

* Windows 10 (22H2 - 19045.4529)
* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.07.05
