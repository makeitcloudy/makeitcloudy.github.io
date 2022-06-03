---
layout: post
title: "How to flash Asus rt-n device with freshtomato"
permalink: "/how-to-flash-asus-rt-n-device-with-freshtomato/"
subtitle: "FreshTomato, Tomato, Open/DD-WRT, Gargoyle, LEDE is popular alternative for SOHO"
cover-img: /assets/img/_cover/img-cover-tomato.jpg
thumbnail-img: /assets/img/_thumb/img-thumb-tomato.jpg
share-img: /assets/img/_cover_/img-cover-tomato.jpg
tags: [HomeLab ,Networking ,FreshTomato]
categories: [HomeLab ,Networking ,FreshTomato]
---
Asus RT-N series is affortable and reliable piece of hardware, which can serve your home network already. The original firmware, can left much to be desired and it may not be very up to date, that's why there are some end users, who decide to flash the device with another piece of firmware, being built by enthusiasts. There are also other devices brands which can digest it, for freshTomato, the Hardware Compatibility List (HCL) is available [here](https://wiki.freshtomato.org/doku.php/hardware_compatibility).

## Prerequisites
+ Your device on the HCL
+ Some carefull planning, to check which version is compatible with your device, by prism of available storage and nvram
+ Windows device (preferably physical one), to run the Asus Restoration Utility

## Howto
*Asus Restoration Utility differs (at least by prism of it's binaries size from router to router), which leads me to the conclusion to use the one which is designated for the hardware used*

+ Asus webpage details, [how to use rescue mode](https://www.asus.com/en/support/FAQ/1000814/)
+ This video also brings a lot of detail, [youtube](https://www.youtube.com/watch?v=_b039vim0Jk) - Update OPENWRT 19.07 to ASUS RT-N16 and bypass ASUS' Firmware Check

1. unplug the device from t he power cord
2. press WPS button, and keep it pressed until the led indicating that the device is up, start to blink (it will blink fast like 2/3 times per second)
3. roughly after 20s release the WPS button
4. wait until the device restars, and boot up again - the wifi control led may be used as your indicator that the device is up again
5. unplug the power cord from the device
6. press the reset button (depending from the device it's location may differ)
7. keep the reset button pressed and at the same time plug the power cord of the device
8. on your device, set the static IP address 192.168.1.10/24
9. run Asus Restoration Utility and use it to send the freshTomato image to the device
10. the image should be sent, roughly in 20s
11. wait until the device gets restarted and load the newest firmware, roughly 90s
12. login to the device https://192.168.1.1 via web browser, login:root password:admin
13. Erase NVRAM: Administration -> Configuration -> Restore Default Configuration -> Erase all data in NVRAM memory (thorough)

Start the device configuration according to your needs.

## Summary
+ This has been sucesfuly tested with RT-N10U, RT-N16U, RT-N18U.
+ FreshTomato [Changelog](https://bitbucket.org/pedro311/freshtomato-arm/src/arm-master/CHANGELOG).
+ The Firmware can be downgraded or upgraded.
+ Bare in mind to erase NVRAM, if this it not done, there are chances for quirks.
That's it.

Last update: 2022.04.14