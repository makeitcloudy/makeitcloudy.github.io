---
full-width: true
layout: post
title: "EUC lab - blog post series"
permalink: "/windows-euc-lab-blog-post-series/"
subtitle: "Container with details for the subsequent blog posts, focused on automated EUC oriented lab setup"
cover-img: /assets/img/cover/img-cover-microsoft.jpg
thumbnail-img: /assets/img/thumb/img-thumb-window.jpg
share-img: /assets/img/cover/img-cover-microsoft.jpg
tags: [HomeLab, Microsoft, DSC]
categories: [HomeLab, Microsoft, DSC]
---
This post is about setting up SQL Server by making use of Powershell and Desired State Configuration, for home lab purposes.


# EUC lab - Blog post series



## Code Details - ADDS



**TODO**: describe the code references and dependecies

The [005_ActiveDirectory_demo.ps1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/005_ActiveDirectory_demo.ps1) script, downloads the [ADDS_setup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/005_ActiveDirectory/ADDS_setup.ps1) to the user profile documents directory.  
* [run_InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/_blogPost/windows-preparation/run_initialSetup.ps1)

* [InitialSetup.ps1](https://raw.githubusercontent.com/makeitcloudy/HomeLab/feature/007_DesiredStateConfiguration/000_targetNode/InitialConfig.ps1)

#### 2.1.1. Code Details - ADDS_setup.ps1

* It contains the DSC script, DSC Configuration is included within the script
* The configuration data is completelly separated from the [ConfigData.psd1](https://github.com/makeitcloudy/HomeLab/blob/feature/007_DesiredStateConfiguration/000_initialConfig/ConfigData.psd1) - this information is important, especially with the IP address modifications of the Domain Controllers, once those are changed, it should be also reflected in the ConfigData.psd1. Otherwise machines won't join to the domain, when the [scenario - Domain Joined VM](https://makeitcloudy.pl/windows-DSC/) from paragraph 2.2 is run

Detailed explanation of the steps to prepare target node (regardless if it is a management or active directory node) is available in the two blog posts

* [windows-preparation](https://makeitcloudy.pl/windows-preparation/) - paragraph 2.0.2
* [windows-dsc](https://makeitcloudy.pl/windows-DSC/) - paragraph

## Code Details - ADCS



```code
.
```

## Code Details - DHCP



```code
.
```

## Code Details - File Services



```code
.
```

### Code Details - File Services - iscsi



```code
.
```

### Code Details - File Services - member server



```code
.
```

### Code Details - SQL 



```code
.
```