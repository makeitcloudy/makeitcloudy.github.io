---
layout: post
title: "OSDBuilder offline servicing updates"
permalink: "/OSDBuilder-offline-servicing-updates/"
subtitle: "Prepare the updated Windows ISO's for the homelab usecase"
cover-img: /assets/img/cover/img-cover-lab.png
thumbnail-img: /assets/img/thumb/img-thumb-open.jpg
share-img: /assets/img/cover/img-cover-lab.png
tags: [HomeLab ,Windows, Imaging]
categories: [HomeLab ,Windows, Imaging]
---
There are plenty of places where the overall process is described, the knowledge is available in other places, here it's being shown in context of the virtual workplace, home lab, regardless whether it is Citrix Virtual Apps and Desktops, Remote Desktop Services or Parallels RAS. The core infrastructure and your compute will benefit out of it, as the patching process of each VM individually is time and resource consuming, especially when you have no orchestration layer for this purpose, and your usecase is home labbing, not an enterprise scale deployments.

If you decide to follow along, you will end up with updated ISO images, which will install in unattended way. If this is combined with the AXL or AutomatedCitrix repo and it's PowerShell XenPLModule available on [github](https://github.com/makeitcloudy/AutomatedCitrix/tree/feature/001_hypervisor).
The module is far from being perfect and it left much to be desired, never the less it is a good starting point, and allows provisioning of VM's directly from PowerShell, without any interaction with the XenCenter or XCP-ng Center.

## Useful Links

+ modern management [youtube](https://www.youtube.com/watch?v=JKE1nrVxheQ)
+ modern management [blog](https://www.moderndeployment.com/quick-start-guide-windows-10-waas-servicing-updates-via-osdbuilder/)
+ makeitcloudy [github](https://github.com/makeitcloudy/AutomatedCitrix/tree/feature/007_imagePrep)
+ manelrodero [github](https://github.com/manelrodero/OSDBuilder)

+ getvpro [blog](https://getvpro.wordpress.com/2022/04/06/custom-offline-iso-windows-deployment-method-as-a-packer-alternative/)
+ getvpro [github](https://github.com/getvpro/Standard-WinBuilds/tree/master/Offline_Builds/Autounattend_xml)

## Prerequisites and observations from the field

+ Dedicated servicing machine (w2k19 or w10), with the Windows ADK installed, I'm not recommending performing the image operations on your management or authoring node. The servicing machines does not have to be joined to the domain but it has to have access to the Internet.
+ Was stuck with servicing machine powered by w10 22H2 - it can **not** access the samba share anymore. Previous versions of the operating system does not have this problem. (smb share - is the same for both)
+ Access to Microsoft Update webpage.
+ Adequate amount of free disk space, roughly each updated operating system consumes somewhere around 20GB of free space  (including the updated iso file).
+ Regardless of the amount of vCPU the overall update process will take roughly up to 4hours. It's more memory intensive and from time to time it bring some level of stress on the storage.
+ Rights to run an elevated PowerShell prompt
+ It would be wise to have a separated drive, where the OSDBuilder is located along with all the media and builds prepared by this great tool
+ If you have more than one image to maintain, due to the fact that the process is time consuming, consider spinning few vms and service the images in parallel.
+ The autounattend.xml for the windows 10, needs an update of the product key (it changes between versions - Enterprise has different KMS key than EnterpriseN - details about the keys can be found [here](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys))

## Download the base image

If you start from zero [MediaCreationTool.bat](https://github.com/AveYo/MediaCreationTool.bat/blob/main/MediaCreationTool.bat) is the tool, which can help you to download the Windows 10.
All you have to do is to rename the .bat file, in my case it was renamed to *enterprise iso 21H2.bat*. You execute it within the elevated PowerShell session. Be prepared that you should have sufficient space on your C: drive (at least 8GB), even though the working directory where you run the MediaCreationTool is on another drive.

## Prepare the OSDBuilder

Before the process can start you need to prepare your VM

1. Install the ADK coresponding to the Operating system of the VM

+ Latest ADK is available [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)
+ Older ADK versions are available [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)

2. Within the installation of the ADK pick only few elements
In case there is a previous version installed, you have uninstall it first, there is no option for a quick update of ADK.

+ Deployment Tools (contains oscdimg.exe - in case your only goal is to burn the updated image, during the ADK installation pick only this option feature from the installation gui)
+ Image and Configuration Designer
+ Configuration Designer
+ User State Migration Tool

Installation of IaCD, CD, USMT, can be skipped if the pure usecase of an update. oscdimg (part of DT) is the dependency for the iso creation once the OSDBuilder does it's trick with the bootable iso preparation.

```powershell
Set-ExecutionPolicy ByPass -Scope CurrentUser
Install-Module OSDBuilder
# store the OSDBuilder within the separate directly aside from your regular OS
Get-OSDBuilder -SetPath O:\OSDBuilder
Get-OSDBuilder -CreatePaths
# nuget along with the untrusted module will be installed from the PS Gallery
Import-Module OSDBuilder
# Download the OneDriveSetup package
# OSDBuilder\Content folder contains all different content which is downloaded like
# one drive, or unattended installation or start menu customization
# eventually instead of 'OneDriveSetup Production' 'OneDriveSetup Enterprise' switch can be used
Get-DownOSDBuilder -ContentDownload 'OneDriveSetup Production'
# Mount the ISO of the media, planed for an update
# for windows 10 iso's it can be done with making use of MediaCreationTool
# https://github.com/AveYo/MediaCreationTool.bat

# In order to get the index of the version which is under your interests, execute
# https://osdbuilder.osdeploy.com/module/functions/import-osmedia
# Import-OSMedia -ShowInfo -Quick

# ImageIndex for Windows Servers editions is the natural number <1-4>
# ImageIndex 1 - standard
# ImageIndex 2 - standard desktop experience
# ImageIndex 3 - datacenter
# ImageIndex 4 - datacenter desktop experience

# ImageIndex for Desktop Operating systems - w10 is the natural number as well
# ImageIndex 1: Windows 10 Education
# ImageIndex 2: Windows 10 Education N
# ImageIndex 3: Windows 10 Enterprise
# ImageIndex 4: Windows 10 Enterprise N
# ImageIndex 5: Windows 10 Pro
# ImageIndex 6: Windows 10 Pro N
# in case it can not enumerate the indexes within the ISO, it may mean that the ISO is corrupted, you need to download it again and start the mount and import again
Import-OSMedia -ImageIndex 1 -SkipGrid -Update -BuildNetFX
# or
Import-OSMedia -ImageName 'Windows 10 Enterprise' -ImageIndex 3 -SkipGrid -Update -BuildNetFX -Verbose

# In this usecase there are no features which are enabled within the image
# It is left as generic as possible, and in case feature or role is needed it's
# getting enabled with making use of DSC (Desired State Configuration) later on
# when managing the configuration drift or with some other mechanisms

# once the whole operations ends it's time to create the ISO
# this command has a dependency on the ADK and the availability of oscdimg.exe
New-OSBMediaISO -FullName 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 1909 18363.2274'
# at this point the updated iso file should be located here
# O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 21H2 19044.2604\ISO

# at this point continue with step number 5 mentioned below
# this section will be updated shortly (2023.02.17 - recalling the process in the lab)
```

The process of the update looks like this:
0. Mount ISO (the regular not updated iso, or the updated iso from the previous update cycle)

1. Import-OSMedia (and pass the Index Parameter, depending from the target (datacenter, standard, desktop experience, core, Enterprise N, Education, Proffessional etc))

2. Update-OSMedia -Name 'name of the folder within the OSBuild folder' which is planned to be updated, similar result can be also achieved with making use of Import-OSMedia with a parameter -Update and -BuildNetFX

3. Once finished (it can take few hours) run the New-OSBMediaISO -FullName 'path to the OSBuilds directory' which was brought as an output from the previous commandlet.

4. Once done unmount the original ISO.

5. At the end repack the ISO with making use of the oscdimg and add the autounattend.xml file [BIOS of UEFI based](https://github.com/makeitcloudy/AutomatedCitrix/tree/feature/007_imagePrep/unattendedISO) or [goodFileIsQuietIsoFile](https://github.com/makeitcloudy/AutomatedCitrix/blob/feature/007_imagePrep/unattendedISO/goodIsoFileIsAquietIsoFile-JohanArwidmark.ps1) prepared by Johan Arwidmark. Alternatively, include the xml directly within the OSDBuilder\Content\Unattend directory with making use of the OSDBuilder author [guide](https://osdbuilder.osdeploy.com/docs/osbuild/content-directory/unattend)

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

Servicing process was tested on vms which were running Windows 10 20H2 and Server 2019 OS'es. Images which were serviced/updated: w10, w2k16, w2k19, w2k22 (depending from the version of the OSDBuilder the list of the operating systems may vary). The .

Last update: 2023.06.01
