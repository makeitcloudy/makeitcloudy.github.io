---
layout: post
title: "Azure tenant and Intune integration"
permalink: "/azure-intune-initial-configuration/"
subtitle: "How to onboard Windows Desktop to Intune"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-azure.png
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, XCP-ng, Windows]
categories: [HomeLab, XCP-ng, Windows]
---
There is no need to buy Intune licenses in order to perform tests in home lab. There is automated lab provisioning way available from Microsoft which is designed to be working on the hyper-v.
It contains few virtual switches which grants the machines access to the internet, plus a bunch of machines which acts as:

+ domain controller
+ gateway server
+ MEM server, which does not have to be used for the Intune usecase
+ bunch of Windows Desktop clients

Materials related to intune available are:
+ Youtube - [Modern Endpoint Manager](http://endpoint-manager.com/)
+ Youtube - [Microsoft Endpoint Manager Commnuity](https://www.youtube.com/@MSEndpointMgr))
+ Dean Ellerby - Udemy - [Intune for Windows Training - includes MD-102 practice exam!](https://www.udemy.com/course/learn-intune/?referralCode=0751E019FC0DD131052C)
+ Travis Roberts, Dean Ellerby - Udemy - [Windows 365 Enterprise and Intune Management](https://www.udemy.com/course/windows-365-enterprise-and-intune-management/)

## Prerequisites

There are two elements which should be bound together in order to make it work.

+ Microsoft 365 Developer program - [apply here](aka.ms/m365devprogram)
+ Windows 11 and Office 365 Deployment Lab Kit - [download here](aka.ms/windows11labkit)

Here is the playlist which explain in detail, how to setup and get access to the hyper-v environment, which is not joined to the domain, which is the majority of usecases for home labbers. [ShotokuTech](https://www.youtube.com/playlist?list=PLVPBipeObwMNm9HpBDCUECL16AG2TMMZU) - explain how to remotelly manage the hyper-v server which is in the workgroup.

An alternative way is to have deployed alike setup on the XCP-ng. You need the following VM's:

+ Active Directory
+ Azure cloud connector which can be shared with other roles
+ Windows Desktop like w10 or w11

## Azure Active Directory - MDM related

The Microsoft 365 Developer program can **not** be bound with existing tenant. It has to be a new tenant, which is provisioned for you for the 90 days by Microsoft. Mine was vpqgt.onmicrosoft.com, though one of the first steps may be adding your own domain and bounding it by making use of the TXT record with the Azure tenant.

```powershell
# 1. Login to azure with the Microsoft 365 developer program login
# 2. Add custom domain to the the exiting tenant
# with DNS Checker you can check the records propagation of your domain
Start-Process 'https://dnschecker.org/#TXT/'
# 3. Azure Active Directory -> Mobility (MDM and MAM) -> Microsoft Intune
# 4. MDM user scope -> All (MAM and MDM can create conflicts, here we make use of the MDM)

# Security defaults - Enable multifator authentication for Admins which is a great starting point
# Enabling security defaults is in contrary with Conditional Access Policies
# 5. Microsoft Entra ID -> Properties -> Manage security details
```

## Windows Enrollment

Windows Enrollment is the first step for managing the device by making use of the configuration profiles. Any supported OS can enroll to intune, though auto-enrollment requires PRO, Enterprise or Education. Microsoft 365 is licensed per user, so is Intune.

+ join
  Active Directory Domain Joined - no record of the device in Azure AD
  Azure Active Directory Domain Joined - cloud only, no record of the device in Azure AD
  Hybrid Azure Active Directory Domain Joined - joined to directly to Active Directory and have it's representation in Azure AD
+ register
  Azure AD Registered
  Not Registered
+ enroll
  Intune Enrolled
  Not Intune Enrolled

### Manual Enrollment

+ Driven by the user
+ Requires Azure AD
+ Requires Intune license
+ Requires CNAME - (not needed when onmicrosoft.com is in use)
+ Supports BYOD

```powershell
# 1. Intune Tenant Admin customizations
https://intune.microsoft.com/
# Once configured properly you should be able to preview the customization on Company Portal
https://portal.manage.microsoft.com/
# 2. Within the policies in the Intune Tenat Admin customizations you can create brainding for the group of users
# 3. CNAME
# Devices -> Enroll Devices -> CNAME Validation
# In case the CNAME validation did not went well go through the 
Start-Process 'https://learn.microsoft.com/en-us/mem/intune/enrollment/windows-enroll#enable-windows-enrollment-without-azure-ad-premium'

# The deployment guide is available here
Start-Process 'https://learn.microsoft.com/en-us/mem/intune/fundamentals/deployment-guide-enrollment-windows'

# Create CNAME
Start-Process 'https://learn.microsoft.com/en-us/mem/intune/enrollment/windows-enrollment-create-cname'

# In your domain administration panel create:
# 3.1 first CNAME record
# Name: EnterpriseEnrollment.[yourdomain]
# TTL: 3600 (1h)
# Value: EnterpriseEnrollment-s.manage.microsoft.com.

# 3.2 second CNAME record
# Name: EnterpriseRegistration.[yourdomain]
# TTL: 3600 (1h)
# Value: EnterpriseRegistration.windows.net.

# Changes to DNS may take up to 72h to propagate, though if your domain is already spread across DNS servers, once you added the CNAME records properly the 
Start-Process 'https://dnschecker.org/#CNAME/EnterpriseEnrollment.[yourdomain]'
# should bring a positive result and the CNAME validation check on
Start-Process 'https://intune.microsoft.com/'
# should be validated properly immediatelly
```

### Install Company Portal on the endpoint

Install company portal on the windows desktop, via Microsoft Store.

At this point:

+ the UPN of 'yourdomain' should be configured on the Active Directory controllers with the domain and trusts properties.
+ the CNAME should be validated on https://intune.microsoft.com
+ the user account should be bound with with the UPN suffix of 'yourdomain'
+ the user account used to login should have an Intune license assigned

Launch the company portal and login with the user.
Check the 'Allow my organization to manage my device'.

### How to recognize if the device is Azure AD joined

On the endpoint navigate to 'your info' and check if it is Local account or Azure AD account.
On the Intune portal navigate to Devices -> All devices, and check if there is a Personal device which matches the name of the device you are manually enrolling.

## Bulk enrollment

Requires AAD Premium (for automatic Enrollment), the m365 developer covers the E5 licenses.
PPKG is sent via email to the end user, which is used for the enrollment.
Edge WebView2 is needed for the provisioning, the WebView2 is also used in the Citrix Workspace App context.

+ [Azure Active Directory Pricing](https://www.microsoft.com/en-us/security/business/microsoft-entra-pricing)
+ [Available versions of Azure AD Multifactor authentication](https://learn.microsoft.com/en-in/entra/identity/authentication/concept-mfa-licensing#available-versions-of-azure-ad-multi-factor-authentication)

```powershell
Start-Process 'https://www.youtube.com/watch?v=rw0rkXfv2pk'
# 1. Login to Azure portal
# 2. Navigate to Microsoft Entra ID
# Check the available features within the m365 developer Entra ID
# Microsoft Entra ID -> Licenses -> Licensed features (available features)

# get the P2
# 3. Entra ID -> Licenses -> All products -> Try/Buy -> Activate Free trial
```

## User licenses

The licenses are the essential piece to bring the functionalities in place.

```powershell
Start-Process 'https://www.microsoft.com/en-us/security/business/microsoft-intune-pricing'
# 1. Entra ID -> Users -> pick user -> Licenses
# if the Microsoft Intune Plan 1 is there you should be covered
```

## Corporate Enrollment

Autoenrollment methods:

+ group policy
+ Azure AD

```powershell
Start-Process 'https://entra.microsoft.com/'
# search on the top for mobility
# 1. Microsoft Intune should be on the list
# 2. Change the MDM user scope from All to Some and define the group name which define who can enroll the devices in your environment (for example for the pilot of devices), though the better way would be to limit that by the licenses assigned to the users, for instance by dynamic group.

# The best practice is prevent to allow the end users to enroll the device and the applications, so the MDM and MAM should NOT be set to ALL.

Start-Process 'https://intune.microsoft.com'
# 1. Devices -> Enroll Devices -> Windows Enrollment -> Automatic Enrollment

# Allow or block personal devices to be auto enrolled (this setting influence the autopilot based enrollment)

# 2. Devices -> Enrollment Device platform restrictions -> All Users -> Properties -> Platform settings -> Edit (here you can define what is the platform which can be enrolled)
#    Personally Owned - Block

# How to check if the device is not BYOD ?
dsregcmd /status

# +--------------------------------------------------------------------------+
# | Device state - those two fields confirmes the Hybrid Azure AD join state |
# +--------------------------------------------------------------------------+
# AzureADJoined : YES
# DomainJoined  : YES - this in majority of cases means that it's corporate owned

# +--------------------------------------------------------------------------+
# | Tenant Details                                                           |
# +--------------------------------------------------------------------------+

# +--------------------------------------------------------------------------+
# | User State                                                               |
# +--------------------------------------------------------------------------+
# NgcSet          : NO - Next Generation Credential - towards Windows Hello
# WorkplaceJoined : NO - The user device is hybrid AzureAD joined

# +--------------------------------------------------------------------------+
# | SSO State                                                                |
# +--------------------------------------------------------------------------+
# AzureADPrt      : Yes - Azure AD Primary Refresh Token Available

# 3. Start Menu -> Manage your account -> Access work or school
#    The key indicator that the device is joined to the MDM is the fact that 
#    it shold contain the info button, just on the left of the disconnect button
#    it is clickable and should show which policies are applied to the device

#    if the Info button is not there it's clear indicator that the device is
#    NOT MDM enrolled
```

## Auto Enrollment

To configure the autoenrollment

```powershell
# 1. Go the the gpmc.msc
# 2. Create a GPO 'Enable Intune AutoEnrollment'
# 3. Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> 
#    MDM -> Enable automatic MDM enrollment using default Azure AD credentials
Start-Process 'https://www.microsoft.com/en-us/download/details.aspx?id=104677'
# 4. Enable | User Credential (the MDM Application is only needed for the device credential)
# 5. Link the policy to the OU with Devices

#    The device credential is only supported for Microsoft Intune enrollment with comanagement for Azure Virtual Desktop, because the Intune subscription is user centric.
Start-Process 'https://techcommunity.microsoft.com/t5/microsoft-intune-blog/understanding-hybrid-azure-ad-join-and-co-management/ba-p/2221201'

# 6. run powershell admin session
gpresult /r /scope:computer
# 7. restart the device

# Now it's time to test whether everything went fine and the device is enrolled

# 8. Start Menu -> Manage your account -> Access work or school
#    The info button should be there.
```

## Enrollment Status Page

```powershell
Stat-Process 'https://intune.microsoft.com'
# 1. Devices -> Enroll Devices -> Enrollment Status Page -> Default profile -> 'All users and devices' -> Settings -> Edit -> 'Show app and profile configuration' -> Yes
```

## Autopilot

Hardware Hash vs Partner Portal

```powershell
Start-Process 'https://www.powershellgallery.com/packages/Get-WindowsAutoPilotInfo/3.5/Content/Get-WindowsAutoPilotInfo.ps1'
#OA3Tool.exe from Windows ADK
$session = New-CimSession
$object = (Get-CimInstance -CimSession $session -Namespace root/cimv2/mdm/dmmap -Class MDM_DevDetail_Ext01 -Filter "InstanceID='Ext' AND ParentID='./DevDetail'")
^ "$env:Programfiles\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Licensing\OA30\oa3tool.exe" /DecodeHwHash="$($devDetail.DeviceHardwareData)"
```

How to get the hardware hash on Windows Desktop

```powershell
Install-Script get-windowsautopilotinfo
Set-ExecutionPolicy bypass
Get-WindowsAutoPilotInfo.ps1
```

## Autopilot - deployment profile

```powershell
Start-Process 'https://intune.microsoft.com'
# 1. Devices -> Windows -> Windows Enrollment -> Deployment Profiles -> Create profile -> Windows PC -> 

# Deployment Mode                  : User-Driven
# Join to Microsoft Entra ID as    : hybrid joined
# Skip AD connectivity check       : NO
# Microsoft Software License Terms : Hide
# Privacy settings                 : Hide
# Hide change account options      : Hide
# User Account type                : Standard
# Allow pre-provision deployment   : NO

# assign the group

# Check
# Devices -> Windows Enrollment -> Devices

# Once enrolled the Intune enforce some policies like the encryption
```

## Conclusions

Last update: 2022.05.31
