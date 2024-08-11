---
full-width: true
layout: post
title: "How to reset routerboard configuration"
permalink: "/how-to-reset-routerboard-configuration/"
subtitle: "It's still the routerboard, but approaches slightly differ between products"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [Networking, Mikrotik]
categories: [Networking, Mikrotik]
---
Even though it is still the routerboard it slightly differ from device to device.
The method works, when there is no access to the device, or you have cut off yourself during the configuration.

## If there is no access

Get the physical access to the device, due to the fact that you cut yourself off from the device, and there is no console port.

### RB951G

1. unplug power cable from device
2. hold reset button
3. plug power cable to device
4. hold reset button only for 2-5 seconds (not 10,20 or 30sec) until ACT LED is flashing - it's enough that it flashes 2 or three times, right after that you'll hear a bip
5. release reset button

### RB951-2n

1. unplug power cable from device
2. hold reset button
3. plug power cable to device
4. hold reset button until the ACT LED starts blinking fast - it's enough that it flashes 5-7 times times
5. release reset button

## If the access is there

If the ssh, L2 or winbox access is there, run following command from the terminal

```bash
/system reset-configuration
#if you like to restore the previous state or different config from the backup
/system reset-configuration run-after-reset=[name of the backup config file]
```

Login to the device via mac or IP address, as the DHCP server is bind to the brige, so ports 2-5 should bring the IP for your device. Ether 1 with default config is being used as your WAN.

## backup existing configuration

```shell
/system backup save name=20220629_mkt951G_config.backup
```

## restore existing configuration

```shell
/system backup load name=20220629_mkt951G_config.backup
```

## Summary

If your Mikrotik has console or serial port, then all what you need is appropriate cable, and usb converter if your device is not equipped with serial port.

Last update: 2022.04.12
