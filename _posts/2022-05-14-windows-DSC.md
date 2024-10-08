---
# https://github.com/daattali/beautiful-jekyll#advanced-parameters
# By default, page content is constrained to a standard width. Use full-width: true to allow the content to span the entire width of the window.
full-width: true
layout: post
title: "Windows management VM - DSC"
permalink: "/windows-DSC/"
subtitle: "Initial Desired State Configuration setup management VM"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
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

## Links

* .

## 0. Assumptions

* Your network device of choice consists a DHCP reservation for the IPv4 Address, until you decide to go with the static IP - for the VM being configured at the moment
* DNS used by the VM can resolve the public Internet addresses
* PowerShell script execution policy is configured properly to run scripts - this was covered during the initial configuration describe in [windows-preparation](https://makeitcloudy.pl/windows-preparation/) blogpost
* winRM service is running on the node - this was configured in the initial configuration as well
* [AutomatedLab](https://github.com/makeitcloudy/AutomatedLab/tree/main) PowerShell Module Installed (otherwise it won't install the DSC modules properly)

## 2. Howto

Run following code in elevated ISE session

```powershell
# how to start an elevated powershell session
cmd
powershell
Start-Process PowerShell -Verb RunAs
```

Run the code from the section which suits particular scenario.

* At this stage there is no domain yet, that's why proceed with the code from paragraph 2.1
* Once the domain is setup, run the code from the paragaph 2.2

### 2.1. Scenario - VM in Workgroup

Run it, when the VM should stay in the workgroup, or there is no Active Directory domain setup yet.

* [run_initialConfigDSC_workgroup.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_workgroupps1)

```powershell
# Start-Process PowerShell_ISE -Verb RunAs
# run in elevated powershell session

# Set-InitialConfigDsc is part of the AutomatedLab module
# it has been downloaded during the preparation steps (when the vmtools were installed)

# at this stage the target node has correct name

#region - Initial Setup - Workgroup
# For succesfull execution Domain does NOT have to be available, DNS should resolve public domains
# Execute this piece of when configuring target node in the Workgroup
Set-InitialConfigDsc -NewComputerName $env:computername -Option Workgroup -Verbose
#endregion

```

### 2.2. Scenario - Domain Joined VM

Run it, when VM should be domain joined.

* [run_initialConfigDSC_domain.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/_blogPost/README.md#run_initialconfigdsc_domainps1)

```powershell
# Set-InitialConfigDsc is part of the AutomatedLab module
# it has been downloaded during the preparation steps (when the vmtools were installed)

# at this stage the target node has correct name - this is covered by the script
# https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfig.ps1
# region 3.5, function Set-NewComputerName - it is a wrapper for the Rename-Computer commandlet which enforce the
# query for the computername itself

# this is important as the mof files are targeted towards nodes based on the name of the vm and it's roles
# for the management node this does not matter that much, though for the subsequent VM's which are
# configured by making use of the same piece of code and logic it affects the proper execution of the code

#region - Initial Setup - Domain
# For succesfull execution Active Directory Domain and DNS should be available
# Execute this piece of when configuring target node, and adding it to the domain

# NOTE: if $domainName variable is modified then the ConfigData.psd1 also has to be ammended accordingly
#https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1
# it may also be that the domain name is hardcoded in some other scripts - figuring this out and ammending is trivial task

$domainName = 'lab.local'  #FIXME
Set-InitialConfigDsc -NewComputerName $env:computername -Option Domain -DomainName $domainName -Verbose
#endregion
```

## 3. Set-InitialConfigDsc.ps1

The Set-InitialConfigDsc.ps1 function leads to the file [InitialConfigDsc.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfigDsc.ps1), stored in GitHub. It references the [ConfigureNode.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureNode.ps1), which has a DSC configuration content split into workgroup and domain usecase. Those are feed in from the [ConfigData.psd1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) file. Once run

### 3.0. What does the code do

* It initialize all variables for succesfull code execution
* It creates the folders structure for the DSC compilations, etc - $env:SYSTEMDRIVE\dsc
* It downloads the powershell functions and configuration
* It downloads [ConfigData.psd1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup
* It downloads [ConfigureLCM.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureLCM.ps1) from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup
* It downloads [ConfigureNode.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigureNode.ps1) from github and store in $env:SYSTEMDRIVE\dsc\config\localhost\InitialSetup
* It installs missing modules for the DSC succesfull execution
* It prepares self signed certificate for securing the credentials
* It takes care about the self signed certificate thumbprint and passes it as a paramter into the LCM configuration
* It configures the LCM
* It starts the actual configuration of the node

It creates [InitialConfigDsc.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfigDsc.ps1) in $env:USERPROFILE\Documents directory.

the target node should have:

* ~~new name, which equals to the $NewComputerName variable~~ - this is covered within the initial configuration, during the execution of [run_initialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1), in the [InitialConfig.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfig.ps1) - section 3.5
* disabled IPv6 address
* ~~defined trusted hosts~~ - it is not needed within this deployment scenario
* WinRM service running

Now the efforts with the [Active Directory DSC](https://makeitcloudy.pl/windows-role-active-directory/) setup, can be made.

## Summary

It was tested on:

* Windows 10 (22H2 - 19045.4529)
* Server 2022 (21H2 - 20348.1547) - Core & Desktop Experience

Last update: 2024.08.05
