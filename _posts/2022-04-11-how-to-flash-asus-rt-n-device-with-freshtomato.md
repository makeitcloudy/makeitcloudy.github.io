---
layout: post
title: "How to flash Asus RT-N device with FreshTomato"
permalink: "/how-to-flash-asus-rt-n-device-with-freshtomato/"
subtitle: "FreshTomato, Tomato, Open/DD-WRT, Gargoyle, LEDE is popular alternative for SOHO"
cover-img: /assets/img/cover/img-cover-tomato.jpg
thumbnail-img: /assets/img/thumb/img-thumb-tomato.jpg
share-img: /assets/img/cover/img-cover-tomato.jpg
tags: [HomeLab ,Networking ,FreshTomato]
categories: [HomeLab ,Networking ,FreshTomato]
---
Asus RT-N series is affortable and reliable piece of hardware, which can serve your home network already. The original firmware, can left much to be desired and it may not be very up to date, that's why there are some end users, who decide to flash the device with another piece of firmware, being built by enthusiasts. There are also other devices brands which can digest it, for freshTomato, the Hardware Compatibility List (HCL) is available [here](https://wiki.freshtomato.org/doku.php/hardware_compatibility).

## Prerequisites

+ Your device on the HCL
+ Some carefull planning, to check which version is compatible with your device, by prism of available storage and nvram
+ Windows device (preferably physical one), to run the Asus Restoration Utility
+ Asus Restoration Utility (which goes hand in hand with the flashed router) installed
+ FreshTomato downloaded and extracted from zip
+ Backup existing configuration of the device, unless you start from scratch

## Download Firmware

+ versions [tomato.groov.pl](https://tomato.groov.pl/?page_id=69)
+ RT-N10u is [K26RT-N](https://freshtomato.org/downloads/freshtomato-mips/2022/2022.6/K26RT-N/Asus%20RT-Nxx%20%26%20CO/) - one of the *freshtomato-K26USB-NVRAM32K_RT-N5x-MIPSR2* releases
+ RN-N18u is [K26ARM](https://freshtomato.org/downloads/freshtomato-arm/2022/2022.6/K26ARM/) - one of the *freshtomato-RT-N18U-ARM_NG* releases

## Howto

*Asus Restoration Utility differs (at least by prism of it's binaries size from router to router), which leads me to the conclusion to use the one which is designated for the hardware used*

+ Asus webpage details, [how to use rescue mode](https://www.asus.com/en/support/FAQ/1000814/)
+ This video also brings a lot of detail, [youtube](https://www.youtube.com/watch?v=_b039vim0Jk) - Update OPENWRT 19.07 to ASUS RT-N16 and bypass ASUS' Firmware Check

0. unplug all devices from the router (ethernet and usb)
1. unplug the router from the power cord
2. press WPS button, and keep it pressed until the led indicating that the device is up, start to blink (it will blink fast like 2/3 times per second)
3. roughly after 20s release the WPS button (after that time the power led starts blinking fast on RTN-18u)
4. wait until the device restars, and boot up again - the wifi control led may be used as your indicator that the device is up again (at this stage the device won't have the class C IP address assigned to it anymore) - be patient it may take a minute or two
5. unplug the power cord from the device
6. press the reset button (depending from the device it's location may differ, for the RTN-18 it is located between the WAN link and the rear USB port)
7. keep the reset button pressed and at the same time plug the power cord of the device (keep it pressed for around 20s and the device will start blinking with the power led - for 15s led is on and then off for the next 15s, this should be an indicator that you can continue. When the router is in this state it should assign to your device, an IP address from class C)
8. reconfigure the network interface of your endpoint, set the static IP address 192.168.1.10/24. You do not need to set the default gateway.

9. run Firmware Restoration aka Asus Restoration Utility (**the version should match the router model**, you won't be fortunate with using restoration utility dedicated to the rtn-16u with rtn-18u device). Use it to send the freshTomato image to the device.

10. the image should be sent, roughly in 20s, at the end of the overall process the Firmware Restoration utility will report the following status: Successfully recovered the system. Please wait for the system to reboot.
11. wait until the device gets restarted and load the newest firmware, roughly 90s (again your wifi led indicator will indicate the device is up again)
12. login to the device http://192.168.1.1 via web browser, 
    login:root <br>
    password:admin <br>
    (in case you can not get there, and the initial 90s already past, unplug the power cord from the device, and power it up again - I have not noticed any pattern with that, never the less it happens occasionally)
13. erase NVRAM: Administration -> Configuration -> Restore Default Configuration -> Erase all data in NVRAM memory (thorough). After proceeding with this step, the total amount of Free NVRAM is 55.65% - 35.62KB/64K.00B (never the less for some reason this might slightly differ between devices)
14. wait while the default are restored (90s)
15. login back to device http://192.168.1.1 via web browser and restore the configuration or start configuring the device according to your needs.
16. reconfigure the network interface of your endpoint, so the network subnets matches

## Summary

+ This has been sucesfuly tested with RT-N10U, RT-N16U, RT-N18U.
+ FreshTomato [Changelog](https://bitbucket.org/pedro311/freshtomato-arm/src/arm-master/CHANGELOG).
+ The Firmware can be downgraded or upgraded.
+ Bare in mind to erase NVRAM, if this it not done, there are chances for quirks.
<br>
That's it.<br>
Last update: 2022.11.27