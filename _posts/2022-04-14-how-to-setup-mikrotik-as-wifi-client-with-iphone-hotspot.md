---
layout: post
title: "How to setup mikrotik as wifi client with iphone hotspot"
permalink: "/how-to-setup-mikrotik-as-wifi-client-with-iphone-hotspot/"
subtitle: "When you are roaming or need a quick backup for your main Internet wire"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik]
categories: [HomeLab ,Networking ,Mikrotik]
---
It's not the most secure way, but can be used as a good starting point for further improvements. It works with iphone and android devices which are your hot spots and the Mikrotik is the client.

## Prerequisites
From what can be read on Mikrotik [forum](https://forum.mikrotik.com/viewtopic.php?t=120218) thread there are some caveats with the mixture of mikrotik and iphone worh exploring.
As it goes for tethering, please have a look into [this](https://forum.mikrotik.com/viewtopic.php?t=79320) thread.

## Rules
+ when your iphone is the client for the Mikrotik (opposite usecase), read the thread, someone is mentioning really interesting thing there. *'we were able to find the problem. It is related to the DHCP lease time.<br>
By default Lease time on the MikroTik DHCP Server is 10 minutes. The iPhone disconnects approx after 5-7 minutes (little more than half of the DHCP lease time).
On TP-link the lease time is 120 minutes and that is why the iPhone is connected a lot longer. If you lower the lease time on the TP-link also to 10 minutes then you will see that it also disconnects in 5-7 minutes.<br>
The solution for this is to increase the DHCP Server lease time to a lot longer time and then iPhone will keep connection to the AP a lot longer.'*
+ (still the opposite usecase) some also says that the country makes a difference, so it's worth trying with the *'united states'* instead of the country where you are living
+ (with the usecase where the Mikrotik is the client for the iphone which is serving the wireless hotspot) I was not very succesfull with the first searches for the wifi. When I enabeld the screen of the device, the scan suddently turned out to be succesfull.
+ once connected, regardless of the state of the screen of the iphone the communication was still in place until the full drain of the battery
+ change your dns settings on the IP -> DNS and on your IP -> DHCP Server -> Networks so you use altenrative to those which your ISP provides you

## Howto
0. plug in the power cord to your device and plug the ethernet cable to one of the ports of the device
1. connect to it via winbox, apply the stock configuration (all your ports will end up on the same LAN bridge, so regardless which ethernet you will be connected to the Mikrotik, management should be in place)
2. Exclude the Wi-Fi interface from the bridge - Bridge -> Ports -> Delete the Wi-Fi interface (wlan1)
3. Include the Ether1 interface in the bridge - Bridge - Ports -> Open ether5 -> Copy -> Change the interface in the new dialog to ether 1 -> OK (this will cause that all your ethernet ports can be used to connect devices, with stock configuration the ether1 is being used as the uplink) - within our scenario we use the wlan1 as the uplink
4. Adjust which is the WAN port - Interfaces -> Interface list - WAN -> wlan1 (with stock it is ether1, it needs to be changed for the wlan1, as it plays the uplink role)
5. Wireless -> WiFi interfaces -> wlan1 -> Mode - Station
5. Connect the Wi-Fi interface as a client to the Wi-Fi network (in our case it is the mobile hotspot from your mobile) - Wireless -> WiFi interfaces -> wlan1 -> Scan -> find the WiFi network and press Connect
6. Adjust the security settings of the WiFi client - Wireless -> Security Profiles -> default -> Mode: dynamic keys; Authentication Types WPA/WPA2 PSK (within SOHO this is the most typical config, actually it also applies to the iphone - but please check the settings on your's as it may differ for some reason). You can also create as many profiles here as you like (depending from how many wifi hotspots you'd like to consume the Internet from, then within the Interface wlan1 -> Wireless -> security profile just change the profile and you will switch from one network to another)
7. Enable the DHCP client on the WiFi interface - IP -> DHCP client -> Change ether1 to wlan1


If everything is OK with the security settings your Mikrotik will connect to the hotspot and the Internet should be reachable.<br>
You can apply further configs on top of that starting for here.

```bash
[user@951G-2Hnd] > /export 
# apr/14/2022 22:49:43 by RouterOS 7.3.1
#
# model = 951G-2HnD
/interface bridge
add admin-mac=(BRIDGE MAC ADDRESS) auto-mac=no comment=defconf name=bridge
/interface list
add comment=defconf name=WAN
add comment=defconf name=LAN
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=951G-2Hnd
add authentication-types=wpa2-psk mode=dynamic-keys name=anotherHotspot supplicant-identity=""
add authentication-types=wpa2-psk mode=dynamic-keys name=iphone supplicant-identity=""
/interface wireless
set [ find default-name=wlan1 ] band=2ghz-b/g/n country=poland disabled=no distance=indoors installation=indoor security-profile=iphone ssid=\
    "iPhone6S" wireless-protocol=802.11
/ip pool
add name=default-dhcp ranges=192.168.88.10-192.168.88.254
/ip dhcp-server
add address-pool=default-dhcp interface=bridge name=defconf
/interface bridge port
add bridge=bridge comment=defconf interface=ether1
add bridge=bridge comment=defconf interface=ether2
add bridge=bridge comment=defconf interface=ether3
add bridge=bridge comment=defconf interface=ether4
add bridge=bridge comment=defconf interface=ether5
/ip neighbor discovery-settings
set discover-interface-list=LAN
/interface list member
add comment=defconf interface=bridge list=LAN
add comment=defconf interface=wlan1 list=WAN
/ip address
add address=192.168.88.1/24 comment=defconf interface=bridge network=192.168.88.0
/ip dhcp-client
add comment=defconf interface=wlan1
/ip dhcp-server network
add address=192.168.88.0/24 comment=defconf dns-server=1.1.1.2,1.1.1.1 gateway=192.168.88.1
/ip dns
set allow-remote-requests=yes
/ip dns static
add address=192.168.88.1 comment=defconf name=router.lan
/ip firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
add action=accept chain=input comment="defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
add action=drop chain=input comment="defconf: drop all not coming from LAN" in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept in ipsec policy" ipsec-policy=in,ipsec
add action=accept chain=forward comment="defconf: accept out ipsec policy" ipsec-policy=out,ipsec
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" connection-state=established,related hw-offload=yes
add action=accept chain=forward comment="defconf: accept established,related, untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=WAN
/ip firewall nat
add action=masquerade chain=srcnat comment="defconf: masquerade" ipsec-policy=out,none out-interface-list=WAN
/ipv6 firewall address-list
add address=::/128 comment="defconf: unspecified address" list=bad_ipv6
add address=::1/128 comment="defconf: lo" list=bad_ipv6
add address=fec0::/10 comment="defconf: site-local" list=bad_ipv6
add address=::ffff:0.0.0.0/96 comment="defconf: ipv4-mapped" list=bad_ipv6
add address=::/96 comment="defconf: ipv4 compat" list=bad_ipv6
add address=100::/64 comment="defconf: discard only " list=bad_ipv6
add address=2001:db8::/32 comment="defconf: documentation" list=bad_ipv6
add address=2001:10::/28 comment="defconf: ORCHID" list=bad_ipv6
add address=3ffe::/16 comment="defconf: 6bone" list=bad_ipv6
/ipv6 firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=input comment="defconf: accept UDP traceroute" port=33434-33534 protocol=udp
add action=accept chain=input comment="defconf: accept DHCPv6-Client prefix delegation." dst-port=546 protocol=udp src-address=fe80::/10
add action=accept chain=input comment="defconf: accept IKE" dst-port=500,4500 protocol=udp
add action=accept chain=input comment="defconf: accept ipsec AH" protocol=ipsec-ah
add action=accept chain=input comment="defconf: accept ipsec ESP" protocol=ipsec-esp
add action=accept chain=input comment="defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
add action=drop chain=input comment="defconf: drop everything else not coming from LAN" in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop packets with bad src ipv6" src-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: drop packets with bad dst ipv6" dst-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: rfc4890 drop hop-limit=1" hop-limit=equal:1 protocol=icmpv6
add action=accept chain=forward comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=forward comment="defconf: accept HIP" protocol=139
add action=accept chain=forward comment="defconf: accept IKE" dst-port=500,4500 protocol=udp
add action=accept chain=forward comment="defconf: accept ipsec AH" protocol=ipsec-ah
add action=accept chain=forward comment="defconf: accept ipsec ESP" protocol=ipsec-esp
add action=accept chain=forward comment="defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
add action=drop chain=forward comment="defconf: drop everything else not coming from LAN" in-interface-list=!LAN
/system clock
set time-zone-name=Europe/Warsaw
/tool mac-server
set allowed-interface-list=LAN
/tool mac-server mac-winbox
set allowed-interface-list=LAN
```

the wireless interface

```bash
[user@951G-2Hnd] > /interface wireless print 
Flags: X - disabled; R - running 
 0  R name="wlan1" mtu=1500 l2mtu=1600 mac-address=8B:B8:BB:91:1B:BC arp=enabled interface-type=Atheros AR9300 mode=station ssid="iPhone6S" 
      frequency=2412 band=2ghz-b/g/n channel-width=20mhz secondary-frequency="" scan-list=default wireless-protocol=802.11 vlan-mode=no-tag vlan-id=1 
      wds-mode=disabled wds-default-bridge=none wds-ignore-ssid=no bridge-mode=enabled default-authentication=yes default-forwarding=yes 
      default-ap-tx-limit=0 default-client-tx-limit=0 hide-ssid=no security-profile=iphone compression=no 
```

the security profiles

```bash
[user@951G-2Hnd] > /interface wireless security-profiles print 
Flags: * - default 
 0 * name="default" mode=none authentication-types="" unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa-pre-shared-key="" wpa2-pre-shared-key="" 
     supplicant-identity="951G-2Hnd" eap-methods=passthrough tls-mode=no-certificates tls-certificate=none mschapv2-username="" mschapv2-password="" 
     disable-pmkid=no static-algo-0=none static-key-0="" static-algo-1=none static-key-1="" static-algo-2=none static-key-2="" static-algo-3=none 
     static-key-3="" static-transmit-key=key-0 static-sta-private-algo=none static-sta-private-key="" radius-mac-authentication=no radius-mac-accounting=no 
     radius-eap-accounting=no interim-update=0s radius-mac-format=XX:XX:XX:XX:XX:XX radius-mac-mode=as-username radius-called-format=mac:ssid 
     radius-mac-caching=disabled group-key-update=5m management-protection=disabled management-protection-key="" 

 1   name="anotherHotspot" mode=dynamic-keys authentication-types=wpa2-psk unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa-pre-shared-key="" 
     wpa2-pre-shared-key="(YOUR WIFI PASSWORD)" supplicant-identity="" eap-methods=passthrough tls-mode=no-certificates tls-certificate=none 
     mschapv2-username="" mschapv2-password="" disable-pmkid=no static-algo-0=none static-key-0="" static-algo-1=none static-key-1="" static-algo-2=none 
     static-key-2="" static-algo-3=none static-key-3="" static-transmit-key=key-0 static-sta-private-algo=none static-sta-private-key="" 
     radius-mac-authentication=no radius-mac-accounting=no radius-eap-accounting=no interim-update=0s radius-mac-format=XX:XX:XX:XX:XX:XX 
     radius-mac-mode=as-username radius-called-format=mac:ssid radius-mac-caching=disabled group-key-update=5m management-protection=disabled 
     management-protection-key="" 

 2   name="iphone" mode=dynamic-keys authentication-types=wpa2-psk unicast-ciphers=aes-ccm group-ciphers=aes-ccm wpa-pre-shared-key="" 
     wpa2-pre-shared-key="(YOUR WIFI PASSWORD)" supplicant-identity="" eap-methods=passthrough tls-mode=no-certificates tls-certificate=none mschapv2-username="" 
     mschapv2-password="" disable-pmkid=no static-algo-0=none static-key-0="" static-algo-1=none static-key-1="" static-algo-2=none static-key-2="" 
     static-algo-3=none static-key-3="" static-transmit-key=key-0 static-sta-private-algo=none static-sta-private-key="" radius-mac-authentication=no 
     radius-mac-accounting=no radius-eap-accounting=no interim-update=0s radius-mac-format=XX:XX:XX:XX:XX:XX radius-mac-mode=as-username 
     radius-called-format=mac:ssid radius-mac-caching=disabled group-key-update=5m management-protection=disabled management-protection-key="" 

```

## Summary
Tested on RouterOS 7.3.1. Device model 951G-2HnD, with Iphone was 6 Plus with the software version 12.5.5. It also should work with ROS 6.X and other Mirkotik devices.<br>
That's it.<br>
Last update: 2022.04.14