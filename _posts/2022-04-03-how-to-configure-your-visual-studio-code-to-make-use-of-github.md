---
full-width: true
layout: post
title: "How to configure your visual studio code to make use of github"
permalink: "/how-to-configure-your-visual-studio-code-to-make-use-of-github/"
subtitle: "How to configure git"
cover-img: /assets/img/cover/img-cover-git.jpg
thumbnail-img: /assets/img/thumb/img-thumb-code.png
share-img: /assets/img/cover/img-cover-git.jpg
tags: [ReadmeFirst, Github, IaC]
categories: [ReadmeFirst, Github, IaC]
---
It's worth putting some attention on making use of git to manage your code and projects, instead of using offline versioning on the filesystem level. When one reach some amount of code, it's just a life saver to move away from chaos. Right, up to some extend ISE will be enough, never the less if you strive towards IaC any features and support from the dev tools and it's extensions, will help a lot.

## Prerequisities

+ Install [Git](https://git-scm.com/downloads)
+ Install [Visual Studio Code](https://code.visualstudio.com/)
+ Install extensions for Visual Studio Code: GitLens - Git supercharged, PowerShell, Azure Resource Manager (ARM) Tools, Azure CLI Tools, Bicep

## Configure Git

+ Run brand new PowerShell console just in case to reinitiate the environmental variables for the console context.
+ Run following commands within your PowerShell console

```powershell
git --version
#setup the user name name and email address
git config --global user.name "Piotr Ostrowski"
git config --global user.email "me@your.domain"
git config --global color.ui true
#at this point your very initial configuration is ready
git config --list
```

+ Create folder on the filesystem which is your local copy of the github repository

```powershell
New-Item -Path $env:SystemDrive\Git -ItemType Directory
Set-Location -Path $env:SystemDrive\Git
```

## Use Git

+ Now you can start cloning your repository (navigate via the web browser to the repository which you'd like to clone locally) like [AutomatedRDS](https://github.com/makeitcloudy/AutomatedRDS) and hit the green icon called 'Code' and copy the link of the repository to clipboard.

```powershell
git clone https://github.com/makeitcloudy/AutomatedRDS.git
#this will result with the information that there is a version controlling within the folder you cloned
Get-ChildItem -Hidden
Set-Location -Path .\AutomatedRDS\
Get-ChildItem .\.git\
```

+ Update cloned repository
+ Stage updates (for instance with the Visual Studio Code GUI)
+ Commit
+ Sync Changes

Starting this point of time your local repository is your single source of truth and this is the place where you make changes and upload them remotelly. When you jump to another device and update your repository

+ Fetch the repository.
+ Pull updates from the repository.
+ Start updating it.
+ Then stage, commit and sync changes.

## Following actions

+ [Setting up Visual Studio Code, GitHub and code signing certificate](https://www.citrixlab.dk/archives/1612)

## Summary

Ps. It may be far from the best practices, and still left much to be desired, never the less you can start from here.
That's it.

Last update: 2022.04.03
