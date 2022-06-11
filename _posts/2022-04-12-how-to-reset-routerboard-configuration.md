---
layout: post
title: "How to reset routerboard configuration"
permalink: "/how-to-reset-routerboard-configuration/"
subtitle: "It's still the routerboard, but approaches slightly differ between products"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik]
categories: [HomeLab ,Networking ,Mikrotik]
---
Even though it is still the routerboard it slightly differ from device to device.
The method works, when there is no access to the device, or you have cut off yourself during the configuration.

# If there is no access
Get the physical access to the device
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

## If the access is there
Run following command from the terminal
```bash
/system reset-configuration
#if you like to restore the previous state or different config from the backup
/system reset-configuration run-after-reset=[name of the backup config file]
```

Login to the device via mac or IP address, as the DHCP server is bind to the brige, so ports 2-5 should bring the IP for your device. Ether 1 with default config is being used as your WAN.
<br>
Last update: 2022.04.12
