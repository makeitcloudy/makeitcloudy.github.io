---
layout: post
title: "How to setup mikrotik vlan - CRS3XX series"
permalink: "/how-to-setup-mikrotik-vlan-crs3xx-series/"
subtitle: "CRS3XX just do it's job pretty well, when it goes for switching"
cover-img: /assets/img/how-to-setup-mikrotik-vlan-crs3xx-series/img-cover.jpg
thumbnail-img: /assets/img/how-to-setup-mikrotik-vlan-crs3xx-series/img-thumb.jpg
share-img: /assets/img/how-to-setup-mikrotik-vlan-crs3xx-series/img-cover.jpg
tags: [HomeLab ,Networking ,Mikrotik]
categories: [HomeLab ,Networking ,Mikrotik]
---
This series of mikrotik devices, can handle your regular VM traffic, as well as your storage traffic. Never the less 10G devices seems to be a better fit for homelab usecase. Not perfect, but good enough.

## Prerequisite
+ CRS3XX device
+ a bit of mikrotik knowledge

## Configure CRS3XX as L2 switch
CRS3XX can act as L3 or L2 device, it's much more effective, when it is being used as a switch and serve your virtualization layer for the communication on the OS level, between all your boxes which constitutes your lab usecase and it's deployment. 
```bash
/interface bridge
add admin-mac=(MAC-ADDRESS) auto-mac=no name=bridge-all-vlans pvid=404 vlan-filtering=yes
/interface ethernet
set [ find default-name=ether1 ] advertise=10M-full,100M-full,1000M-full comment="nas - em0"
set [ find default-name=ether2 ] advertise=10M-full,100M-full,1000M-full comment="nas - em1"
set [ find default-name=ether3 ] advertise=10M-full,100M-full,1000M-full comment="nas - em2"
set [ find default-name=ether4 ] disabled=yes
set [ find default-name=ether5 ] advertise=10M-full,100M-full,1000M-full comment="node3 - eth1"
set [ find default-name=ether6 ] advertise=10M-full,100M-full,1000M-full comment="node3 - eth2"
set [ find default-name=ether7 ] advertise=10M-full,100M-full,1000M-full comment="node3 - eth3"
set [ find default-name=ether8 ] advertise=10M-full,100M-full,1000M-full comment="node3 - eth4"
set [ find default-name=ether9 ] advertise=10M-full,100M-full,1000M-full comment="node2 - eth1"
set [ find default-name=ether10 ] advertise=10M-full,100M-full,1000M-full comment="node2 - eth2 - admit only tagged"
set [ find default-name=ether11 ] advertise=10M-full,100M-full,1000M-full comment="node2 - eth3 - admin only tagged"
set [ find default-name=ether12 ] advertise=10M-full,100M-full,1000M-full comment="node2 - eth4 - admin only tagged"
set [ find default-name=ether13 ] advertise=10M-full,100M-full,1000M-full comment="node1 - eth1"
set [ find default-name=ether14 ] advertise=10M-full,100M-full,1000M-full comment="node1 - eth2"
set [ find default-name=ether15 ] advertise=10M-full,100M-full,1000M-full comment="node1 - eth3"
set [ find default-name=ether16 ] advertise=10M-full,100M-full,1000M-full comment="node1 - eth4"
set [ find default-name=ether17 ] advertise=10M-full,100M-full,1000M-full comment="CRS3XX - bond - CCR - eth8"
set [ find default-name=ether18 ] advertise=10M-full,100M-full,1000M-full comment="CRS3XX - bond - CCR - eth7"
set [ find default-name=ether19 ] advertise=10M-full,100M-full,1000M-full comment="CRS3XX - bond - CCR - eth6"
set [ find default-name=ether20 ] advertise=10M-full,100M-full,1000M-full comment="CRS3XX - bond - CCR - eth5"
set [ find default-name=ether21 ] advertise=10M-full,100M-full,1000M-full comment="node0 - eth1"
set [ find default-name=ether22 ] advertise=10M-full,100M-full,1000M-full comment="node0 - eth2"
set [ find default-name=ether23 ] advertise=10M-full,100M-full,1000M-full comment="node0 - eth3"
set [ find default-name=ether24 ] advertise=10M-full,100M-full,1000M-full comment="node0 - eth4"
set [ find default-name=sfp-sfpplus1 ] advertise=10M-full,100M-full,1000M-full disabled=yes
set [ find default-name=sfp-sfpplus2 ] advertise=10M-full,100M-full,1000M-full disabled=yes
/interface vlan
add interface=bridge-all-vlans name=vlan404LabMgmt vlan-id=404
/interface bonding
add min-links=1 mode=802.3ad name=bond-20-19-18-17 slaves=ether20,ether19,ether18,ether17 transmit-hash-policy=layer-2-and-3
/interface list
add name=WAN
add name=LAN
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/user group
set full policy=local,telnet,ssh,ftp,reboot,read,write,policy,test,winbox,password,web,sniff,sensitive,api,romon,dude,tikapp
/interface bridge port
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether1 pvid=1005
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether2 pvid=2006
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether3 pvid=3007
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether4 pvid=4008
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether5 pvid=1005
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether6 pvid=2006
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether7 pvid=3007
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether8 pvid=4008
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether9 pvid=1005
add bridge=bridge-all-vlans frame-types=admit-only-vlan-tagged interface=ether10 pvid=2006
add bridge=bridge-all-vlans frame-types=admit-only-vlan-tagged interface=ether11 pvid=3007
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-vlan-tagged interface=ether12 pvid=4008
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether13 pvid=1005
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether14 pvid=2006
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether15 pvid=3007
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether16 pvid=4008
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether17 pvid=404
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether18 pvid=404
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether19 pvid=404
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether20 pvid=404
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether21 pvid=1005
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether22 pvid=2006
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether23 pvid=3007
add bridge=bridge-all-vlans frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=ether24 pvid=4008
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=sfp-sfpplus1
add bridge=bridge-all-vlans disabled=yes frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes interface=sfp-sfpplus2
add bridge=bridge-all-vlans interface=bond-20-19-18-17 pvid=404
/ip neighbor discovery-settings
set discover-interface-list=!none
/interface bridge vlan
add bridge=bridge-all-vlans comment="bridge vlan4008 - node 4th int" tagged=bridge-all-vlans,bond-20-19-18-17 untagged=ether24,ether16,ether12,ether8,ether4 \
    vlan-ids=4008
add bridge=bridge-all-vlans comment="bridge vlan3007 - node 3rd int" tagged=bridge-all-vlans,bond-20-19-18-17 untagged=ether23,ether15,ether11,ether7,ether3 \
    vlan-ids=3007
add bridge=bridge-all-vlans comment="bridge vlan2006 - node 2nd int" tagged=bridge-all-vlans,bond-20-19-18-17 untagged=ether22,ether14,ether10,ether6,ether2 \
    vlan-ids=2006
add bridge=bridge-all-vlans comment="bridge vlan1005 - node 1st int" tagged=bridge-all-vlans,bond-20-19-18-17 untagged=ether21,ether13,ether9,ether5,ether1 \
    vlan-ids=1005
add bridge=bridge-all-vlans comment="bridge vlan404 - mgmt - it transports the mgmt IP address of the CRS3XX switch" tagged=bridge-all-vlans,bond-20-19-18-17 vlan-ids=404
add bridge=bridge-all-vlans comment="bridge vlan2010 - maybe some storage traffic - nothing connected" disabled=yes tagged=bond-20-19-18-17 untagged=sfp-sfpplus2 vlan-ids=4094
add bridge=bridge-all-vlans comment="bridge vlan1010 - maybe some storage traffic - nothing connected" disabled=yes tagged=bond-20-19-18-17 untagged=sfp-sfpplus1 vlan-ids=4093
/interface list member
add disabled=yes interface=ether1 list=LAN
add disabled=yes interface=ether2 list=LAN
add disabled=yes interface=ether3 list=LAN
add disabled=yes interface=ether4 list=LAN
add disabled=yes interface=ether5 list=LAN
add disabled=yes interface=ether6 list=LAN
add disabled=yes interface=ether7 list=LAN
add disabled=yes interface=ether8 list=LAN
add disabled=yes interface=ether9 list=LAN
add disabled=yes interface=ether10 list=LAN
add disabled=yes interface=ether11 list=LAN
add disabled=yes interface=ether12 list=LAN
add disabled=yes interface=ether13 list=LAN
add disabled=yes interface=ether14 list=LAN
add disabled=yes interface=ether15 list=LAN
add disabled=yes interface=ether16 list=LAN
add disabled=yes interface=ether17 list=LAN
add disabled=yes interface=ether18 list=LAN
add disabled=yes interface=ether19 list=LAN
add disabled=yes interface=ether20 list=LAN
add disabled=yes interface=ether21 list=LAN
add disabled=yes interface=ether22 list=LAN
add disabled=yes interface=ether23 list=LAN
add disabled=yes interface=ether24 list=LAN
add disabled=yes interface=sfp-sfpplus1 list=LAN
add disabled=yes interface=sfp-sfpplus2 list=LAN
/ip address
add address=192.168.88.1/24 comment=defconf disabled=yes interface=bridge-all-vlans network=192.168.88.0
/ip dhcp-client
add disabled=no interface=vlan404LabMgmt
/ip dns
set servers=(YOUR-DNS-SERVERS)
/ip dns static
add address=(IP-ADDRESS-OF-YOUR-NODE) name=nas
/ip firewall service-port
set ftp disabled=yes
set tftp disabled=yes
set irc disabled=yes
set h323 disabled=yes
set sip disabled=yes
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set ssh port=(SSH-PORT)
set api disabled=yes
set api-ssl disabled=yes
/ip smb
set allow-guests=no
/ip ssh
set forwarding-enabled=remote
/system clock
set time-zone-autodetect=no time-zone-name=Europe/Warsaw
/system identity
set name=CRS3XX
/system leds
set 0 disabled=yes interface=ether1 leds=ether1-led type=interface-speed-1G
set 1 disabled=yes interface=ether2 leds=ether2-led type=interface-speed-1G
set 2 disabled=yes interface=ether3 leds=ether3-led type=interface-speed-1G
set 3 disabled=yes interface=ether4 leds=ether4-led type=interface-speed-1G
set 4 disabled=yes interface=ether5 leds=ether5-led type=interface-speed-1G
set 5 disabled=yes interface=ether6 leds=ether6-led type=interface-speed-1G
set 6 disabled=yes interface=ether7 leds=ether7-led type=interface-speed-1G
set 7 disabled=yes interface=ether8 leds=ether8-led type=interface-speed-1G
set 8 disabled=yes interface=ether9 leds=ether9-led type=interface-speed-1G
set 9 disabled=yes interface=ether10 leds=ether10-led type=interface-speed-1G
set 10 disabled=yes interface=ether11 leds=ether11-led type=interface-speed-1G
set 11 disabled=yes interface=ether12 leds=ether12-led type=interface-speed-1G
set 12 disabled=yes interface=ether13 leds=ether13-led type=interface-speed-1G
set 13 disabled=yes interface=ether14 leds=ether14-led type=interface-speed-1G
set 14 disabled=yes interface=ether15 leds=ether15-led type=interface-speed-1G
set 15 disabled=yes interface=ether16 leds=ether16-led type=interface-speed-1G
set 16 disabled=yes interface=ether17 leds=ether17-led type=interface-speed-1G
set 17 disabled=yes interface=ether18 leds=ether18-led type=interface-speed-1G
set 18 disabled=yes interface=ether19 leds=ether19-led type=interface-speed-1G
set 19 disabled=yes interface=ether20 leds=ether20-led type=interface-speed-1G
set 20 interface=ether21 leds=ether21-led type=interface-speed-1G
set 21 disabled=yes interface=ether22 leds=ether21-led type=interface-speed-1G
set 22 disabled=yes interface=ether23 leds=ether22-led type=interface-speed-1G
set 23 disabled=yes interface=ether24 leds=ether23-led type=interface-speed-1G
set 24 disabled=yes leds="" type=on
set 25 disabled=yes interface=sfp-sfpplus1 leds=sfp-sfpplus1-led1
set 26 disabled=yes interface=sfp-sfpplus2 type=interface-speed
/system ntp client
set enabled=yes primary-ntp=(PRIMARY-NTP-SERVER) secondary-ntp=(SECONDARY-NTP-SERVER)
/system package update
set channel=long-term
/system routerboard settings
set boot-os=router-os
/tool bandwidth-server
set enabled=no
/tool mac-server
set allowed-interface-list=none
```

## Summary
That's it.
Last update: 2022.04.13