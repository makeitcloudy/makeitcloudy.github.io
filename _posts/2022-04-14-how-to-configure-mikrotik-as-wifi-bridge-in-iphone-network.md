---
layout: post
title: "Mikrotik wifi bridge for Apple developers"
permalink: "/how-to-configure-mikrotik-as-wifi-bridge-in-iphone-network/"
subtitle: "Mikrotik acts as the interface converter, your dev machine in the same network range as your iphone"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik]
categories: [HomeLab ,Networking ,Mikrotik]
---
It's not the most secure way, but can be used as a good starting point for further improvements. It works with iphone and android devices which are your hot spots and the Mikrotik is the client. Mikrotik acts as interface converter and bridge, no routing functionality. Mobile is your hotspot, RouterBoard along with all devices connected to the ethenet ports are your iphone clients.

## A bit of background

One of the blog reader Arturus had a usecase, where he needs to get his dev device within the same network range as the iphone. There is a server within the the network, along with the dev device connected via LAN cables to the devices, no masquerade, all devices served via IP addresses comming from the iphone. The network mask is 255.255.255.240 on the iphone hotspot, which means that 13 devices + mikrotik should be able to be handled by the iOS.<br>
In case there are L2 protocols which are used to communicate between devices, and the mikrotik should play the role of the interface converter, or one is willing to use cable connectivity, it may be the solution for you.

By default the iOS 15.7 serves the 172.20.10.1/28.

## Caveats

I've seen it disconnecting from the mobile, once the display of the device turn off. I'm sure there is some configuraton within the mobile, to get this sorted out.

## Prerequisites

+ development machine, or dev server within the same network segment
+ Iphone with iOS 15.7 or never, never the less it should also works with older OSes like 12.X

## Configuration

I'm sure there are better ways to achieve similar functionality, if this is the case reach out towards MUM or colleagues who shares the Mikrotik knowledge on youtube.

1. perform the backup of your existing configuration, and export it to safe place.

2. unplug the power cord out of your device
3. press the res button and plug the power cord to the device, until it's configuration is reset to the default (the device will be playing the switch role, no routing functionality)
4. connect to the device via winbox, and during the initial connection choose to option **not** to use any default configuration
5. connect to the device via it's MAC address (if the connectivity if flapping close all windows within the Winbox or press CTRL+D it should help)
At this stage there should be no bridges, firewall, DHCP client or server configured
6. to configure your Mikrotik as bridge/interface converter following content

```shell
[user@951G-2Hnd] > /export 
# nov/13/2022 19:47:08 by RouterOS 6.49.6
#
# model = 951G-2HnD
/interface bridge
add name=bridgeAllPort
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
add authentication-types=wpa2-psk,wpa2-eap mode=dynamic-keys name=iphone6s supplicant-identity="" wpa2-pre-shared-key="(yourIphoneWi-Fi Password from Personal Hostpot)"
/interface wireless
set [ find default-name=wlan1 ] band=2ghz-b/g/n channel-width=20/40mhz-XX country=poland disabled=no frequency=auto installation=indoor mode=\
    station-pseudobridge security-profile=iphone6s ssid="" wps-mode=disabled
/interface bridge port
add bridge=bridgeAllPort interface=wlan1
add bridge=bridgeAllPort interface=ether1
add bridge=bridgeAllPort interface=ether2
add bridge=bridgeAllPort interface=ether3
add bridge=bridgeAllPort interface=ether4
add bridge=bridgeAllPort interface=ether5
/ip neighbor discovery-settings
set discover-interface-list=!dynamic
/ip dhcp-client
add disabled=no interface=bridgeAllPort
/system clock
set time-zone-name=Europe/Warsaw
```

## Configuration - security profiles

The hotspot on your mobile is password protected, that's why it has to be reflected within the Mikrotik configuration.

```shell
[user@951G-2Hnd] > /interface wireless security-profiles print 
Flags: * - default 
 0 * name="default" mode=none authentication-types="" unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa-pre-shared-key="" wpa2-pre-shared-key="" 
     supplicant-identity="MikroTik" eap-methods=passthrough tls-mode=no-certificates tls-certificate=none mschapv2-username="" mschapv2-password="" 
     disable-pmkid=no static-algo-0=none static-key-0="" static-algo-1=none static-key-1="" static-algo-2=none static-key-2="" static-algo-3=none 
     static-key-3="" static-transmit-key=key-0 static-sta-private-algo=none static-sta-private-key="" radius-mac-authentication=no radius-mac-accounting=no 
     radius-eap-accounting=no interim-update=0s radius-mac-format=XX:XX:XX:XX:XX:XX radius-mac-mode=as-username radius-called-format=mac:ssid 
     radius-mac-caching=disabled group-key-update=5m management-protection=disabled management-protection-key="" 

 1   name="iphone6s" mode=dynamic-keys authentication-types=wpa2-psk,wpa2-eap unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa-pre-shared-key="" 
     wpa2-pre-shared-key="(yourIphoneWi-Fi Password from Personal Hostpot)" supplicant-identity="" eap-methods=passthrough tls-mode=no-certificates tls-certificate=none mschapv2-username="" 
     mschapv2-password="" disable-pmkid=no static-algo-0=none static-key-0="" static-algo-1=none static-key-1="" static-algo-2=none static-key-2="" 
     static-algo-3=none static-key-3="" static-transmit-key=key-0 static-sta-private-algo=none static-sta-private-key="" radius-mac-authentication=no 
     radius-mac-accounting=no radius-eap-accounting=no interim-update=0s radius-mac-format=XX:XX:XX:XX:XX:XX radius-mac-mode=as-username 
     radius-called-format=mac:ssid radius-mac-caching=disabled group-key-update=5m management-protection=disabled management-protection-key="" 
```

## Configuration - wireless interface

There is nothing to configure here apart from the security-profile section.

```shell
[user@951G-2Hnd] > /interface wireless print 
Flags: X - disabled, R - running 
 0    name="wlan1" mtu=1500 l2mtu=1600 mac-address=8B:B8:BB:91:1B:BC arp=enabled interface-type=Atheros AR9300 mode=station-pseudobridge ssid="" frequency=auto 
      band=2ghz-b/g/n channel-width=20/40mhz-XX secondary-frequency="" scan-list=default wireless-protocol=any vlan-mode=no-tag vlan-id=1 wds-mode=disabled 
      wds-default-bridge=none wds-ignore-ssid=no bridge-mode=enabled default-authentication=yes default-forwarding=yes default-ap-tx-limit=0 
      default-client-tx-limit=0 hide-ssid=no security-profile=iphone6p compression=no 
```

## Summary

Tested on RouterOS 6.49.6. Device model 951G-2HnD, with Iphone was 6s with the software version 15.7. It also should work with ROS 7.X and other Mirkotik devices.

That's it.

Last update: 2022.11.13
