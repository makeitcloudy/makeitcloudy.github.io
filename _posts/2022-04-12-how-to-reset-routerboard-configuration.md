---
layout: post
title: "How to setup mikrotik for home lab usecase"
permalink: "/how-to-reset-routerboard-configuration/"
subtitle: "It's still the routerboard, but approaches slightly differ between products"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking]
categories: [HomeLab ,Networking]
---
Even though it is still the routerboard it slightly differ from device to device.

## RB951G
1. unplug power cable from device
2. hold reset button
3. plug power cable to device
4. hold reset button only for 2-5 seconds (not 10,20 or 30sec) until ACT LED is flashing - it's enough that it flashes 2 or three times, right after that you'll hear a bip
5. release reset button

## RB951-2n
1. unplug power cable from device
2. hold reset button
3. plug power cable to device
4. hold reset button until the ACT LED starts blinking fast - it's enough that it flashes 5-7 times times
5. release reset button

Login to the device via mac or IP address, as the DHCP server is bind to the brige, so ports 2-5 should bring the IP for your device. Ether 1 with default config is being used as your WAN.

Last update: 2022.04.12
