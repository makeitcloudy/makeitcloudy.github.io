---
layout: post
title: "How to setup vlan - Mikrotik RouterBoard and CCR series"
permalink: "/how-to-setup-mikrotik-vlan-routerboard-and-CCR-series/"
subtitle: "There are plenty of alternative vendors, but still as of now, mikrotik it's cost effective"
cover-img: /assets/img/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/how-to-setup-mikrotik-vlan-routerboard-and-ccr-series/img-thumb.jpg
share-img: /assets/img/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik]
categories: [HomeLab ,Networking ,Mikrotik]
---
RouterBoard is sufficient for most use cases. It can be the heart of your home lab, unless you pass great amount of traffic which utilize the who throughput of the wires, then there are better product series, like CCR or CRSXXX.<br>
The logic mentioned within this article also applies to CCR series (at least as it goes for the VLAN configuration).

## Prerequisites
+ Mikrotik device from RouterBoard series - in this case RB951-2n (the configuration won't change if one use another product from RB series)
+ There is also an option to virtualize your networking layer by setting up EVE-NG, TheNetworkBerg is detailing the setup on his [youtube](https://www.youtube.com/watch?v=Nl0xKw7RGww) video.

## Rules
+ There is great change you will cut off yourself from the device, if you do not take care about the order. Comparing to the CCR or CRS series, you have to be much carefull with the configuration steps. Use the safe mode within winbox, save the configuration frequently, when you progress with it, so it is easily possible to revert to it's previous state when you make the pitfall. There is no serial port, sometimes your only option is to reset the config to it's defaults and restore the configuration from backup, to it's previous state.
+ When you reset the device to its default configuration, and you apply the initial configuration comming from the vendor, it applies the firewall configuration. Bare in mind that within the configuration showed in this blog post, you need to take care about the Interface list assignments. This configuration will guarantee that you get access to manage the device from the vlan1342 and vlan1345, which are bound to the ether2 and ether5. If this is not in place then the firewall rule which is assigned by default to the device, will drop those frames, and you won't get access.<br>
By default you can see that someone who wrote the initial script which is being served to you takes care about the bridge level access, so you are save until the point of time, when you get rid of the PVID1 from the bridge. Frequenly, those devices are just plugged in, and someone assign the DHCP on top of the bridge (which also seems to be the default configuration), and all devices connected to the RouterBoard via the ethernet ports are within the same broadcast domain, and you can manage the device via winbox, when you are connected to any of the RB ports within that bridge.

```bash
[user@rb951-2n] /interface list> export 
# apr/13/2022 22:37:52 by RouterOS 6.48.6
#
# model = 951-2n
/interface list
add comment=defconf name=WAN
add comment=defconf name=LAN
/interface list member
add comment=defconf interface=bridge-all-vlans list=LAN
add comment=defconf interface=vlan70uplink list=WAN
add interface=vlan1342 list=LAN
add interface=vlan1345 list=LAN
```

I'm sure there are better ways (firewall rules) never the less, this way (by assigning a VLAN being a member of LAN list) you can also control whether the devices connected to particular vlan can or can not. The VLANS assigned to the LAN list - will be granted to have a chance to manage the device.
The following rule is controlling this

```bash
[user@rb951-2n] /interface list>/ip firewall filter export 
add action=drop chain=input comment="defconf: drop all not coming from LAN" in-interface-list=!LAN
```

It works with combination of IP services, but still the firewall rule takes precedence, which means that if something is not allowed on the firewall level, networks configured on *ip services* wont change the behaviour. Those needs to go hand in hand, maybe it's even better not to confiugre any ranges here, but keep the rules within the firewall.

```bash
[user@rb951-2n] /interface list>/ip service export 
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
#this rule allows you getting access from those subnets over ssh
set ssh address=10.0.134.0/24,10.2.134.0/24,10.3.134.0/24,10.4.134.0/24,10.5.134.0/24,10.44.134.0/24
set api disabled=yes
#tihs rule alows you getting access using winbox
set winbox address=10.0.134.0/24,10.2.134.0/24,10.3.134.0/24,10.4.134.0/24,10.5.134.0/24,10.44.134.0/24
set api-ssl disabled=yes
```

+ DHCP Server / Networks - Network, **Netmask** is crucial, use the CIDR notation. The winbox interface allows you configuring it without the netmask entry, it wont throw any error. **My experience is that it won't assign the IP Addresses properly towards the devices connected within the vlans, until the netmask field is configured properly**. The DHCP servers runs on vlan interfaces.

```bash
[user@rb951-2n] /interface list>/ip dhcp-server network export 
add address=10.5.134.0/24 dns-server=1.1.1.1,1.1.1.2 gateway=10.5.134.254 netmask=24
```

+ Strive towards the configuration where you get the access to manage the device over winbox or ssh, from one of the VLANS, once you get there, you are good to go with getting rid of the PVID 1 from the bridge, and configure the VLAN on the last interface which gave you control to manage the device. *(This condition applies when you are connecting to the mikrotik device, using the IP, not MAC - with MAC connectivity, there is some tip to prevent from frequent disconnections, somewhere on the NetworkBerg channel can not recall the fix now)*.
+ When you run PVID 1 on your bridge, then one of the DHCP servers is running on the bridge Interface as well, and all remainig DHCP servers which are meant to serve the clients connected to the ethernet interfaces of the device runs on top of VLAN interfaces.
+ The modification of the bridge PVID needs to be done last, otherwise there is great chance of a cut off.
+ Correct me if I am wrong, never the less the hardware offloading with VLANs wont work with RB series, when it acts as L3 device. This assumption comes from the Mikrotik [table](https://wiki.mikrotik.com/wiki/Manual:Interface/Bridge#Bridge_Hardware_Offloading) and Bridge VLAN Filtering column

```bash
[user@rb951-2n] > /interface ethernet switch print 
Flags: I - invalid 
 #   NAME      TYPE             MIRROR-SOURCE MIRROR-TARGET     SWITCH-ALL-PORTS
 0   switch1   Atheros-7240     none                            none
```

as well as the fact that when you assign the PVID, the H dissapears from here (which means to me that the CPU is handling the traffic and there is no switch chip offload, to release the efforts from the CPU). If you need it, use the CRS3XX series.

```bash
[user@rb951-2n] > /interface bridge port print
Flags: X - disabled, I - inactive, D - dynamic, H - hw-offload 
 #     INTERFACE    BRIDGE            HW  PVID PRIORITY  PATH-COST INTERNAL-PATH-COST    HORIZON
 0 I   ;;; defconf
       ether2      bridge-all-vlans   yes 1342     0x80         10                 10       none
 1     ;;; defconf
       ether3      bridge-all-vlans   yes 1343     0x80         10                 10       none
 2 I   ;;; defconf
       ether4     bridge-all-vlans    yes 1344     0x80         10                 10       none
 3     ;;; defconf
       ether5     bridge-all-vlans    yes 1345     0x80         10                 10       none
 4 XI   ;;; defconf
       wlan1      bridge-all-vlans           1     0x80         10                 10       none
 5     ;;; uplink
       ether1     bridge-all-vlans    no    70     0x80         10                 10       none
```

and the configuration export

```bash
[user@rb951-2n] > /interface bridge port export
/interface bridge port
add bridge=bridge-all-vlans comment=defconf interface=ether2 pvid=1342
add bridge=bridge-all-vlans comment=defconf interface=ether3 pvid=1343
add bridge=bridge-all-vlans comment=defconf interface=ether4 pvid=1344
add bridge=bridge-all-vlans comment=defconf interface=ether5 pvid=1345
add bridge=bridge-all-vlans comment=defconf disabled=yes interface=wlan1
add bridge=bridge-all-vlans comment=uplink hw=no interface=ether1 pvid=70
```

+ if you are interested about the CPU utilization it can be quickly checked, by making use of tools -> profile, it is much more specific that the system resource.

```bash
/tools profile duration=30
/system resource print
```

+ ip routes - I've stumble upon some quirks that the gateway for the Dynamic route was not correct, for some reason it was automatically pointing to the .1 when in my network topology it was .254.

```bash
[user@rb951-2n] > /ip route print 
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip, b - bgp, o - ospf, m - mme, B - blackhole, U - unreachable, P - prohibit 
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE
 0 A S  0.0.0.0/0                          10.0.70.254               1
 1  DS  0.0.0.0/0                          10.0.70.254               1
 2 ADC  10.0.70.0/24       10.0.70.253     vlan70uplink              0
 3 ADC  10.0.134.0/24      10.0.134.254    bridge-all-vlans          0
 4 ADC  10.2.134.0/24      10.2.134.254    vlan1342                  0
 5 ADC  10.3.134.0/24      10.3.134.254    vlan1343                  0
 6 ADC  10.4.134.0/24      10.4.134.254    vlan1344                  0
 7 ADC  10.5.134.0/24      10.5.134.254    vlan1345                  0
```

+ ip routes - due to the fact that it was that inconvenience with the wrong gateway there was a manual configuration done so the Dst. Address 0.0.0.0/0 goes via gateway 10.0.70.254. In the code below you can also observe, that one extra route is added for one of the networks reachable via OpenVPN tunnel.

```bash
[user@rb951-2n] > /ip route export
/ip route
add distance=1 gateway=10.0.70.254
add comment="network which due to the topology can not be recognized automatically and you need to configure it manually" distance=1 dst-address=172.16.134.0/24 gateway=ovpn
```

+ ip firewall service - if there is a VoIP phone, among the devices connected the SIP-ALG should be disabled, to make the SIP streams working properly.

```bash
[user@rb951-2n] > /ip firewall service-port export 
/ip firewall service-port
set sip sip-direct-media=no
```

+ once you have a running configuration, it is a great material to study which elements are the prerequisites for the further blocks built on top of that. If you try configuring things the other way around, or you simply do know the relations between those, it's much harder to progress.

## Howto
I'm sure there better ways achieving this result, never the less this configuration worked for me.
+ on top of it you can put your firewall rules, as well as try to harden the device configuration, as per the best practices shared on the Internet.
+ apart from vlans there are also other virtual interface which can be configured on the device like ppoe, ovpn.

```bash
[user@rb951-2n] > /export 
# apr/13/2022 22:37:52 by RouterOS 6.48.6
# model = 951-2n
/interface bridge
add admin-mac=(MAC-ADDRESS-OF-THE-BRIDGE) auto-mac=no comment=defconf name=bridge-all-vlans pvid=1345 vlan-filtering=yes
/interface wireless
set [ find default-name=wlan1 ] band=2ghz-b/g/n channel-width=20/40mhz-XX country=poland distance=indoors frequency=auto installation=indoor mode=ap-bridge \
    ssid=rb951-2n wireless-protocol=802.11
/interface ethernet
set [ find default-name=ether1 ] comment=uplink
/interface vlan
add interface=bridge-all-vlans name=vlan70uplink vlan-id=70
add disabled=yes interface=bridge-all-vlans name=vlan404mgmt vlan-id=404
add interface=bridge-all-vlans name=vlan1342 vlan-id=1342
add interface=bridge-all-vlans name=vlan1343 vlan-id=1343
add interface=bridge-all-vlans name=vlan1344 vlan-id=1344
add interface=bridge-all-vlans name=vlan1345 vlan-id=1345
/interface list
add comment=defconf name=WAN
add comment=defconf name=LAN
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/ip pool
add name=dhcp_bridge-all-vlans ranges=10.0.134.200-10.0.134.253
add name=dhcp_vlan1342 ranges=10.2.134.201-10.2.134.253
add name=dhcp_vlan1343 ranges=10.3.134.201-10.3.134.253
add name=dhcp_vlan1344 ranges=10.4.134.201-10.4.134.253
add name=dhcp_vlan1345 ranges=10.5.134.201-10.5.134.253
add name=dhcp_vlan404 ranges=10.44.134.201-10.44.134.253
/ip dhcp-server
add address-pool=dhcp_bridge-all-vlans interface=bridge-all-vlans lease-time=12h name=dhcpServer-bridge
add address-pool=dhcp_vlan404 interface=vlan404mgmt lease-time=2h name=dhcp-vlan404
add address-pool=dhcp_vlan1342 disabled=no interface=vlan1342 lease-time=8h name=dhcp-vlan1342
add address-pool=dhcp_vlan1343 disabled=no interface=vlan1343 lease-time=8h name=dhcp-vlan1343
add address-pool=dhcp_vlan1344 disabled=no interface=vlan1344 lease-time=8h name=dhcp-vlan1344
add address-pool=dhcp_vlan1345 disabled=no interface=vlan1345 lease-time=8h name=dhcp-vlan1345
/interface bridge port
add bridge=bridge-all-vlans comment=defconf interface=ether2 pvid=1342
add bridge=bridge-all-vlans comment=defconf interface=ether3 pvid=1343
add bridge=bridge-all-vlans comment=defconf interface=ether4 pvid=1344
add bridge=bridge-all-vlans comment=defconf interface=ether5 pvid=1345
add bridge=bridge-all-vlans comment=defconf disabled=yes interface=wlan1
add bridge=bridge-all-vlans comment=uplink hw=no interface=ether1 pvid=70
/ip neighbor discovery-settings
set discover-interface-list=LAN
/interface bridge vlan
add bridge=bridge-all-vlans disabled=yes untagged=bridge-all-vlans,ether4 vlan-ids=404
#vlan70 is tagged on ether1 which is the uplink
add bridge=bridge-all-vlans tagged=bridge-all-vlans,ether1 vlan-ids=70
#if the uplink is untagged then the ether1 should be untagged
#add bridge=bridge-all-vlans tagged=bridge-all-vlans untagged=ether1 vlan-ids=70
add bridge=bridge-all-vlans tagged=bridge-all-vlans untagged=ether2 vlan-ids=1342
add bridge=bridge-all-vlans tagged=bridge-all-vlans untagged=ether3 vlan-ids=1343
add bridge=bridge-all-vlans tagged=bridge-all-vlans untagged=ether4 vlan-ids=1344
add bridge=bridge-all-vlans tagged=bridge-all-vlans untagged=ether5 vlan-ids=1345
/interface list member
add comment=defconf interface=bridge-all-vlans list=LAN
add comment="this is for the usecase when your uplink serves you tagged vlan" interface=vlan70uplink list=WAN
add comment="this allows the vlan1342 to manage the device via winbox and ssh, if you remove this, devices from this vlan can not do it anymore" interface=\
    vlan1342 list=LAN
add comment="this allows the vlan1345 to manage the device via winbox and ssh" interface=vlan1345 list=LAN
/ip address
add address=10.0.134.254/24 comment="bridge - LAN" interface=bridge-all-vlans network=10.0.134.0
add address=10.44.134.254/24 disabled=yes interface=vlan404mgmt network=10.44.134.0
add address=10.2.134.254/24 interface=vlan1342 network=10.2.134.0
add address=10.3.134.254/24 interface=vlan1343 network=10.3.134.0
add address=10.4.134.254/24 interface=vlan1344 network=10.4.134.0
add address=10.5.134.254/24 interface=vlan1345 network=10.5.134.0
/ip dhcp-client
add comment=uplink disabled=no interface=vlan70uplink use-peer-dns=no use-peer-ntp=no
/ip dhcp-server lease
add address=10.5.134.253 client-id=1:(DEVICE-MAC-ADDRESS) comment=device mac-address=(DEVICE-MAC-ADDRESS) server=dhcp-vlan1345
/ip dhcp-server network
add address=10.0.134.0/24 comment=\
    "the bridge vlan PVID has been changed from 1, so it goes hand in hand with the VLAN ID on the ether port, this entry can be removed" dns-server=1.1.1.2,1.1.1.3 gateway=10.0.134.254 netmask=24 ntp-server=149.156.70.60,54.37.233.160,91.212.242.21,162.159.200.1
add address=10.44.134.0/24 comment="as the vlan404 is disabled, this entry can be removed" gateway=10.44.134.254 netmask=24
add address=10.5.134.248/32 comment="static reservation for - Cisco SPA502G - with different set of DNS servers" dns-server=8.8.8.8,8.8.4.4 gateway=10.5.134.254 netmask=24 ntp-server=46.175.224.7,213.199.225.40
add address=10.2.134.0/24 dns-server=1.1.1.3,1.1.1.2 gateway=10.2.134.254 netmask=24 ntp-server=149.156.70.60,54.37.233.160,91.212.242.21,162.159.200.1
add address=10.3.134.0/24 dns-server=1.1.1.3,1.1.1.2 gateway=10.3.134.254 netmask=24 ntp-server=149.156.70.60,54.37.233.160,91.212.242.21,162.159.200.1
add address=10.4.134.0/24 dns-server=1.1.1.3,1.1.1.2 gateway=10.4.134.254 netmask=24
add address=10.5.134.0/24 dns-server=1.1.1.1,1.0.0.1 gateway=10.5.134.254 netmask=24
/ip dns
set allow-remote-requests=yes servers=1.1.1.2,1.1.1.3
/ip dns static
add address=10.0.45.254 comment=defconf name=router.lan
/ip firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
add action=accept chain=input comment="defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
add action=drop chain=input comment="defconf: drop all not coming from LAN" in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept in ipsec policy" ipsec-policy=in,ipsec
add action=accept chain=forward comment="defconf: accept out ipsec policy" ipsec-policy=out,ipsec
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" connection-state=established,related
add action=accept chain=forward comment="defconf: accept established,related, untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=WAN
/ip firewall nat
add action=masquerade chain=srcnat comment="defconf: masquerade" ipsec-policy=out,none out-interface-list=WAN
/ip firewall service-port
set ftp disabled=yes
set tftp disabled=yes
set irc disabled=yes
set sip sip-direct-media=no
set pptp disabled=yes
set dccp disabled=yes
set sctp disabled=yes
/ip route
add distance=1 gateway=10.0.70.254
add comment="network which due to the topology can not be recognized automatically and you need to configure it manually" distance=1 dst-address=172.16.134.0/24 gateway=ovpn
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set ssh address=10.0.134.0/24,10.2.134.0/24,10.3.134.0/24,10.4.134.0/24,10.5.134.0/24,10.44.134.0/24
set api disabled=yes
set winbox address=10.0.134.0/24,10.2.134.0/24,10.3.134.0/24,10.4.134.0/24,10.5.134.0/24,10.44.134.0/24
set api-ssl disabled=yes
/system clock
set time-zone-name=Europe/Warsaw
/system identity
set name=rb951-2n
/system ntp client
set enabled=yes primary-ntp=46.175.224.7 secondary-ntp=213.199.225.40 server-dns-names=1.0.0.1,1.1.1.1
/system package update
set channel=long-term
/tool bandwidth-server
set enabled=no
/tool mac-server
set allowed-interface-list=LAN
/tool mac-server mac-winbox
set allowed-interface-list=LAN
```

## Summary
This was tested on RouterOS 6.48.6. Device model 951-2n.<br>
That's it.<br>
Last update: 2022.04.13