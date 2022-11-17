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

+ Dedicated servicing machine, with the Windows ADK installed, I'm not recommending performing the image operations on your management or authoring node. The servicing machines does not have to be joined to the domain but it has to have access to the Internet.
+ Access to Microsoft Update webpage.
+ Adequate amount of free disk space, roughly each updated operating system consumes somewhere around 20GB of free space  (including the updated iso file).
+ Regardless of the amount of vCPU the overall update process will take roughly up to 4hours. It's more memory intensive and from time to time it bring some level of stress on the storage.
+ Rights to run an elevated PowerShell prompt
+ It would be wise to have a separated drive, where the OSDBuilder is located along with all the media and builds prepared by this great tool

## Prepare the OSDBuilder

Before the process can start you need to prepare your VM

1. Install the ADK coresponding to the Operating system of the VM

+ Latest ADK is available [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)
+ Older ADK versions are available [here](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)

2. Within the installation of the ADK pick only few elements
In case there is a previous version installed, you have uninstall it first, there is no option for a quick update of ADK.

+ Deployment Tools
+ Image and Configuration Designer
+ Configuration Designer
+ User State Migration Tool

It seems some of those can be trimmed for the pure usecase of an update, but those four will do the trick. Actually the most important thing seems to be the oscdimg which is being used to prepare the bootable ISO.

```powershell
Set-Execution ByPass
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

# ImageIndex for the servers is the natural number <1-4>, for windows 10 it could be <1,10>
# 1 - standard
# 2 - standard desktop experience
# 3 - datacenter
# 4 - datacenter desktop experience
# in case it can not enumerate the indexes within the ISO, it may mean that the ISO is corrupted, you need to download it again and start the mount and import again
Import-OSMedia -ImageIndex 1 -SkipGrid -Update -BuildNetFX

# In this usecase there are no features which are enabled within the image
# It is left as generic as possible, and in case feature or role is needed it's
# getting enabled with making use of DSC (Desired State Configuration) later on
# when managing the configuration drift or with some other mechanisms

# once the whole operations ends it's time to create the ISO
New-OSBMediaISO -FullName 'O:\OSDBuilder\OSBuilds\Windows 10 Enterprise x64 1909 18363.2274'
```
The process of the update looks like this
0. Mount ISO (the regular not updated iso, or the updated iso from the previous update cycle)
1. Import-OSMedia (and pass the Index Parameter, depending from the target (datacenter, standard, desktop experience, core, Enterprise N, Education, Proffessional etc))
2. Update-OSMedia -Name 'name of the folder within the OSBuild folder' which is planned to be updated, similar result can be also achieved with making use of Import-OSMedia with a parameter -Update and -BuildNetFX
3. Once finished (it can take few hours) run the New-OSBMediaISO -FullName 'path to the OSBuilds directory' which was brought as an output from the previous commandlet.
4. Once done unmount the original ISO.

5. At the end repack the ISO with making use of the oscdimg and add the autounattend.xml file [BIOS of UEFI based](https://github.com/makeitcloudy/AutomatedCitrix/tree/feature/007_imagePrep/unattendedISO) or include the xml directly within the OSDBuilder\Content\Unattend directory with making use of the OSDBuilder author [guide](https://osdbuilder.osdeploy.com/docs/osbuild/content-directory/unattend)

## Summary

That's it.

If everything went well the based image/images should contain latest updates along with prefered customizations allowing unattended installations.

Tested on Windows 10 20H2, servicing box. Images services: w10, w2k16, w2k19, w2k22. It is just doing it's trick properly.

Last update: 2022.04.22