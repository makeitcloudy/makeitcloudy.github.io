---
full-width: true
layout: post
title: "OSDBuilder offline servicing updates"
permalink: "/OSDBuilder-offline-servicing-updates/"
subtitle: "Prepare the updated Windows ISO's for the homelab usecase"
cover-img: /assets/img/cover/img-cover-lab.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-lab.jpg
tags: [HomeLab, Windows, OSD]
categories: [HomeLab, Windows, OSD]
---
If you decide to follow along, you will end up with updated ISO images, which will install in unattended way.

## Useful Links

+ modern management [youtube](https://www.youtube.com/watch?v=JKE1nrVxheQ)
+ modern management [blog](https://www.moderndeployment.com/quick-start-guide-windows-10-waas-servicing-updates-via-osdbuilder/)
+ makeitcloudy [github](https://github.com/makeitcloudy/AutomatedCitrix/tree/feature/007_imagePrep)
+ manelrodero [github](https://github.com/manelrodero/OSDBuilder)

+ getvpro [blog](https://getvpro.wordpress.com/2022/04/06/custom-offline-iso-windows-deployment-method-as-a-packer-alternative/)
+ getvpro [github](https://github.com/getvpro/Standard-WinBuilds/tree/master/Offline_Builds/Autounattend_xml)

## Prerequisites and observations from the field

+ Dedicated servicing machine (w2k19 or w10), with the Windows ADK installed, I'm not recommending performing the image operations on your management or authoring node. The servicing machines does not have to be joined to the domain but it has to have access to the Internet
+ If you have more than one OS to service, use more than one VM to service the image, f.e if you have four images - create four VM's and run the actions in parallel
+ Have a separated drive for the OSDBuilder
+ Adequate amount of free disk space, roughly each updated operating system consumes somewhere around 20GB of free space  (including the updated iso file).
+ Regardless of the amount of vCPU the overall update process will take roughly up to 4hours. It's more memory intensive and from time to time it bring some level of stress on the storage.
+ Rights to run an elevated PowerShell prompt, you run the whole process under the elevated session
+ The autounattend.xml for the windows 10, needs an update of the product key (it changes between versions - Enterprise has different KMS key than EnterpriseN - details about the keys can be found [here](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys))

## Download the base image

+ [MediaCreationTool.bat](https://github.com/AveYo/MediaCreationTool.bat/blob/main/MediaCreationTool.bat)
+ rename the .bat file, f.e *enterprise iso 21H2.bat*. You can execute it within the elevated PowerShell session. Be prepared that you should have sufficient space on your C: drive (at least 8GB), even though the working directory where you run the MediaCreationTool is on another drive.

## Prepare the OSDBuilder

Before the process can start you need to prepare your VM

1. Install the ADK coresponding to the Operating system of the VM

[Latest ADK](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install)
[Older ADK versions](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)

2. Within the installation of the ADK pick only few elements
In case there is a previous version installed, you have uninstall it first, there is no option for a quick update of ADK.

+ Deployment Tools (contains oscdimg.exe - in case your only goal is to burn the updated image, during the ADK installation pick only this option feature from the installation gui)

The components below can be used, though are not necessary if the sole purpose is to burn the iso, and autounattend.xml is already prepared
+ Image and Configuration Designer
+ Configuration Designer
+ User State Migration Tool

```powershell
#region authoringBox - initial preparation 
#region authoringBox - ADK installation
# https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install
# Install only Deployment Tools
#endregion

#region authoringBox - initializing disk O for the ISO storage

#endregion

#region authoringBox - OSDBuilder installation

Set-ExecutionPolicy ByPass -Scope CurrentUser -Force
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
Install-Module OSDBuilder -Force
Install-Module OSDSUS -Force
# OSDBUilder 23.2.21.1 | OSD 24.6.18.1
# store the OSDBuilder within the separate directly aside from your regular OS
Get-OSDBuilder -SetHome O:\OSDBuilder
Get-OSDBuilder -CreatePaths

Get-OSDBuilder
## Update OSDBuilder
OSDBuilder -UpdateModule
OSDBuilder -Update
Update-OSDSUS
#endregion
#endregion

Import-Module -Name OSDBuilder -Force
Import-Module -Name OSDSUS -Force
```

## The update process

The process of the update consists of:

0. Mount ISO (the regular not updated iso, or the updated iso from the previous update cycle)
1. Import-OSMedia (and pass the Index Parameter, depending from the target (datacenter, standard, desktop experience, core, Enterprise N, Education, Proffessional etc))
2. unmount the ISO
3. Update-OSMedia -Name 'name of the folder within the OSBuild folder' which is planned to be updated, similar result can be also achieved with making use of Import-OSMedia with a parameter -Update and -BuildNetFX
4. Once finished (it can take few hours) run the New-OSBMediaISO -FullName 'path to the OSBuilds directory' which was brought as an output from the previous commandlet.

### windows 10

1. download the evaluation is from the Internet
2. store it in the $isoFolder directory

```powershell
# Tested 2024.06

$isoName = '10_21H1_LTSC.iso'
$isoFolder = 'O:\ISO\notUpdated\w10\evalCenter'
$isoSourcePath = Join-Path -Path $isoFolder -ChildPath $isoName

$imageName = 'Windows 10 Enterprise N LTSC Evaluation x64 21H2 19044.4529'
$taskName = '202406_w2k22_eval_desktopExperience_updates'

Mount-DiskImage -ImagePath $isoSourcePath -Verbose

# 2. Import iso
# ImageIndex 1: Windows 10 Enterprise LTSC Evaluation
# ImageIndex 2: Windows 10 Enterprise N LTSC Evaluation
Import-OSMedia -Index 2

# Unmount Diskimage

Get-OSMedia -GridView -Verbose

#Update-OSMedia -Download -Execute
# OS Media does not have the latest WSUSXML (MS Updates)
# Use the following command before furnning New-OSBuild
Update-OSMedia -Name $imageName -Download -Execute

# Now that the OS Media is imported and updated, you must create a task that you’ll use to re-run the monthly updates with any features you enable or disable.
# Run this command to create a new task (or re-run a previously created task). 
# In my case, I’m planning to enable .NET 3.5 SP1 so I’ve created the task name as shown below. 
# Create the name that makes sense for you.
New-OSBuildTask -TaskName $taskName -EnableNetFX3

New-OSBuild -Download -Execute -ByTaskName $taskName
#pick the OSImport

# copy the autounatted.xml here
Invoke-Item -Path 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise N LTSC Evaluation x64 21H2 19044.4529\OS'

Get-OSBuilds

New-OSBMediaISO -FullName 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise N LTSC Evaluation x64 21H2 19044.4529'
#New-OSBMediaISO -FullName "O:\OSDBuilder\OSBuilds\$windowsServer2022ReleaseName"
#pick OSBuild - (pushed up all updates, and enabled netfx)

#ISO should be stored in following directory:
Invoke-Item -Path "O:\OSDBuilder\OSBuilds\$imageName\ISO"

#Copy the image to your ISO repository
```

### windows server

1. download the evaluation is from the Internet
2. store it in the $isoFolder directory
3. repeat the process twice (core and desktop exeprience)

```powershell
$w2k22_eval_iso_path = 'O:\ISO\notUpdated\server\2022\w2k22.iso'
$imageName = 'Windows Server 2022 Datacenter Evaluation Desktop Experience x64 21H2 20348.587'
$taskName = '202406_w2k22_eval_desktopExperience_updates'

Mount-DiskImage -ImagePath $w2k22_eval_iso_path

#ImageIndex 1: Windows Server 2022 Standard Evaluation
#ImageIndex 2: Windows Server 2022 Standard Evaluation (Desktop Experience)
#ImageIndex 3: Windows Server 2022 Datacenter Evaluation
#ImageIndex 4: Windows Server 2022 Datacenter Evaluation (Desktop Experience)
Import-OSMedia -Index 4

# Unmount Diskimage

Get-OSMedia
#$windowsServer2022Release = 'Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587'
#Get-OSMedia | Where-Object {($_.Name -match $windowsServer2022ReleaseName) -and ($_.Revision -eq 'OK') -and ($_.Updates -eq 'Update')} |
#              Foreach {Update-OSMedia -Download -Execute -Name $_.Name}

New-OSBuildTask -TaskName $taskName -EnableNetFX3
New-OSBuild -Download -Execute -ByTaskName $taskName
#pick the OSImport

# copy the autounattend.xml to the directory
Invoke-Item -Path 'O:\OSDBuilder\OSBuilds\Windows Server 2022 Datacenter Evaluation Desktop Experience x64 21H2 20348.587\OS'

New-OSBMediaISO
#New-OSBMediaISO -FullName "O:\OSDBuilder\OSBuilds\$windowsServer2022ReleaseName"
#pick OSBuild - (pushed up all updates, and enabled netfx)

#ISO should be stored in following directory:
Invoke-Item -Path "O:\OSDBuilder\OSBuilds\$imageName\ISO"

# Copy the image to your ISO repository
# rename copied is to w10ent_21H2_2406_untd_bios
```

### oscdimg

run an elevated CMD and execute

```code
cd C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg
oscdimg.exe -bootdata:2#p0,e,b"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg\etfsboot.com"#pEF,e,b"C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg\efisys_noprompt.bin" -u2 -udfver102 "O:\ISO\updated_OSDBuilder_result\2022\Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587" "O:\ISO\updated_Final_result\2022\Windows Server 2022 Datacenter Evaluation x64 21H2 20348.587\w2k22dtc_2406_updt_untd_bios.iso"

```

## Unattended Iso - UEFI context

For some reason the UEFI worked for me during the updates at 2023 February, though at 2024 June, can not repeat the process anymore, and receving UEFI shell instead of the regular instlalation process. Never the less, the guides how to get rid of the press any key message for the UEFI boots can be found here.

[BIOS of UEFI based](https://github.com/makeitcloudy/AutomatedCitrix/tree/feature/007_imagePrep/unattendedISO) or [goodFileIsQuietIsoFile](https://github.com/makeitcloudy/AutomatedCitrix/blob/feature/007_imagePrep/unattendedISO/goodIsoFileIsAquietIsoFile-JohanArwidmark.ps1) prepared by Johan Arwidmark. Alternatively, include the xml directly within the OSDBuilder\Content\Unattend directory with making use of the OSDBuilder author [guide](https://osdbuilder.osdeploy.com/docs/osbuild/content-directory/unattend)

```powershell
<#
.Synopsis
    Sample script for Deployment Research
    For UEFI Deployments, modifies a WinPE ISO to not ask for "Press Any Key To Boot From..."

.DESCRIPTION
    Created: 2020-01-10
    Version: 1.0
     
    Author : Johan Arwidmark
    Twitter: @jarwidmark
    Blog   : https://deploymentresearch.com
 
    Disclaimer: This script is provided "AS IS" with no warranties, confers no rights and 
    is not supported by the author or DeploymentArtist..

.NOTES
    Requires Windows ADK 10 to be installed

.EXAMPLE
    N/A
#>

# Settings
$architecture = "amd64" # Or x86
$adkPath = "${env:ProgramFiles(x86)}\Windows Kits\10\Assessment and Deployment Kit"
#$WinPE_ADK_Path = $adkPath + "\Windows Preinstallation Environment"
$oscdimgPath = $adkPath + "\Deployment Tools" + "\$architecture\Oscdimg"

$isoPath                        = 'O:\iso'
$isoUpdatedFolderName           = 'updated_2302'
$isoUpdatedUnattendedFolderName = 'updated_2302_unattended'
$isoUpdatedFileName             = 'w10_21H2_updt_2302.iso'
$isoFinalFileName               = 'w10_21H2_updt_2302_unattended_noprompt.iso'

#path where the OSDBuilder output iso is stored
$isoUpdatedPath           = Join-Path -Path $isoPath -ChildPath $isoUpdatedFolderName
#location of the iso which is processed to become unattended, no prompt
$isoUpdatedFile           = Join-Path -Path $isoUpdatedPath -ChildPath $isoUpdatedFileName

#path where the final iso file is stored
$isoUpdatedUnattendedPath = Join-Path -Path $isoPath -ChildPath $isoUpdatedUnattendedFolderName 
#location of the iso which is the outcome of the script processing
$isoUpdatedUnattendedFile = Join-Path -Path $isoUpdatedUnattendedPath -ChildPath $isoFinalFileName

# Validate locations
If (!(Test-path $isoUpdatedFile)){ Write-Warning "ISO file updated by OSDBuilder does not exist, aborting...";Break}
If (!(Test-path $adkPath)){ Write-Warning "ADK Path does not exist, aborting...";Break}
If (!(Test-path $oscdimgPath)){ Write-Warning "OSCDIMG Path does not exist, aborting...";Break}

# Mount the Original ISO ($isoUpdatedFile) and figure out the drive-letter
Mount-DiskImage -ImagePath $isoUpdatedFile
$isoImage = Get-DiskImage -ImagePath $isoUpdatedFile | Get-Volume
$isoDrive = [string]$isoImage.DriveLetter+":"

# Section added for BIOS Support
# the updated iso content is copied to the Temp Directory
# this steps is to add the autounattend.xml file and repack the iso later on again
$TempFolder = "O:\Temp\ISO"
New-Item -Path $TempFolder -ItemType Directory -Force
Copy-Item "$ISODrive\*" $TempFolder -Recurse
Remove-Item (Join-Path $TempFolder "boot\bootfix.bin") -Force

# Copy the unattended file
# (update 2023.02)this section needs some improvement and autodetection of the OS type or an argument passed into the script or function once it is integrated with the module
# at this point the autounattend.xml file should already exist somewhere on the filesystem

# pick this file if the iso is server based
$autounattendFileName = 'autounattend.xml'
# NOTE: the xml file for the windows 10 i slightly different that for the server operating systems
$autounattendFullPath = Join-Path -Path $isoPath -ChildPath $autounattendFileName
Copy-Item $autounattendFullPath -Destination $TempFolder -Verbose

# Dismount the Original ISO
Dismount-DiskImage -ImagePath $isoUpdatedFile

# Create a new bootable  ISO file, based on the Original ISO, but using efisys_noprompt.bin instead
$BootData='2#p0,e,b"{0}"#pEF,e,b"{1}"' -f "$oscdimgPath\etfsboot.com","$oscdimgPath\efisys_noprompt.bin"
   
$Proc = Start-Process -FilePath "$oscdimgPath\oscdimg.exe" -ArgumentList @("-bootdata:$BootData",'-u2','-udfver102',"$TempFolder\","`"$isoUpdatedUnattendedFile`"") -PassThru -Wait -NoNewWindow
if($Proc.ExitCode -ne 0)
{
    Throw "Failed to generate ISO with exitcode: $($Proc.ExitCode)"
}
else {
    #remove the content of the tempfolder so the the logic can be repeated with the subsequent image or OS version
    Remove-Item $TempFolder -Force
}
```

## Summary

That's it.

If everything went well the based image should contain latest updates ready for unattended installation.

Servicing process was tested on vms which were running Windows 10 20H2,21H2, Server 2019, 2022 H1 OS'es. Images which were serviced/updated: w10, w2k16, w2k19, w2k22 (depending from the version of the OSDBuilder the list of the operating systems may vary). 

Last update: 2024.06.20
