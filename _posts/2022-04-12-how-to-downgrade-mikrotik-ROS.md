---
full-width: true
layout: post
title: "How to downgrade Mikrotik Router OS"
permalink: "/how-to-downgrade-mikrotik-ros/"
subtitle: "When you consider changing branches"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [Networking, Mikrotik]
categories: [Networking, Mikrotik]
---
At the time of writing this blog post the long term release of the Router OS is 6.48.6, where the stable version of ROS 7 is 7.3.1. Downgrade from 7.X to 6.X requires downloading the .npk package from the mikrotik website.

## Background

1. Identify the model, CPU architecute and the installed version of Router OS
2. Downgrade / Upgrade Router OS
3. Downgrade / Upgrade firmware

+ there is great chance that at least some of your configuration will still work after the ROS downgrade/upgrade
+ users and configuration remains
+ backups of the configuration remains on the filesystem

## Downgrade

This section describes how to downgrade the Mikrotik Router OS and firmware.

### Downgrade from ROS 7.X to 6.X

When for some reason your device should be on ROS 6.X but you aready upgraded to ROS 7.X

1. Figure out what is the model of your mikrotik device
2. Figure out it's architecture (ARM,ARM64,MIPSBE,MMIPS,SMIPS,TILE,PPC,X86,General)

```shell
## version shows the installed version of the ROS
## the CPU section is showing it's architecture
## for 951G it's MIPS
/system resource print
## confirm the version of the firmware
/system routerboard print
```

If you are still not sure navigate to the [Support & Downloads](https://mikrotik.com/product/RB951G-2HnD#fndtn-downloads) section of the webpage, and hit the Download of the RouterOS current release. Name of the file will determine the architecture. In context of 951G it is mipsbe.
3. Download the ROS version which goes hand in hand with your CPU architecture from [mikrotik website](https://mikrotik.com/download)
4. Upload the .npk file into the mikrotik filesystem (for instance via the winbox) Files->Upload
5. Execute following command

```shell
## after confirming the wish to make it happen the device will restart automatically
/system package downgrade
```

6. Update the firmware, correct, updating the firmware will actually downgrade it. The firmware upgrade should be done after the RouterOS upgrade/downgrade.

```shell
/system routerboard upgrade
```

7. Restart your device

```shell
/system reboot
```

8. Confirm that the Router OS and the RouterBoard firmware is 

```shell
## check the version of the Router OS
/system resource print
## check the version of the firmware
/system routerboard print
```

## Upgrade

This section describe how to upgrade Router OS and firmware.

### Upgrade to latest version in current branch

```shell
/system package update print
/system package update set channel=upgrade
/system package update check-for-updates
/system package update download
## the update process last ~1min and should end with a reboot in case it is not
/system reboot
## upgrade firmware
/system routerboard upgrade
```

### Upgrade from ROS 6.X to 7.X

```shell
/system package update print
/system package update set channel=testing
/system package update check-for-updates
/system package update download
## the update process last ~1min and should end with a reboot in case it is not
/system reboot
## upgrade firmware
/system/routerboard/print
/system/routerboard/upgrade
/system/reboot
```

## Summary

Tested on 951G,951-2n,CCR.

Last update: 2022.04.12
