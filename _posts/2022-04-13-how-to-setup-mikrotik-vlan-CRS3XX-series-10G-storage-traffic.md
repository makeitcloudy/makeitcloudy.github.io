---
layout: post
title: "How to setup mikrotik vlan - CRS3XX series - 10G storage traffic"
permalink: "/how-to-setup-mikrotik-vlan-crs3xx-series-10G-storage-traffic/"
subtitle: "10G series CRS3XX just do it's job pretty well, when it goes for switching, also for storage traffic"
cover-img: /assets/img/how-to-setup-mikrotik-vlan-crs3xx-series-10G-storage-traffic/img-cover.jpg
thumbnail-img: /assets/img/how-to-setup-mikrotik-vlan-crs3xx-series-10G-storage-traffic/img-thumb.jpg
share-img: /assets/img/how-to-setup-mikrotik-vlan-crs3xx-series-10G-storage-traffic/img-cover.jpg
tags: [HomeLab ,Networking ,Mikrotik]
categories: [HomeLab ,Networking ,Mikrotik]
---
This series of mikrotik devices, can handle your regular VM traffic, as well as your storage traffic. Never the less 10G devices seems to be a better fit for homelab usecase. Not perfect, but good enough.

## Prerequisite
+ CRS3XX 10G device
+ a bit of mikrotik knowledge

## Configure CRS3XX as L2 switch
CRS3XX can act as L3 or L2 device, it's much more effective, when it is being used as a switch and serve your storage traffic, between all your nodes which serves the virtualization layer. The overall gain is very much visible, when you provision the machines with Citrix MCS (Machine Creation Services), or take benefit from the shared storage.
```bash
/interface ethernet
set [ find default-name=ether1 ] comment="CRS3XX - mgmt"
set [ find default-name=sfp-sfpplus1 ] comment="node4 - NAS" l2mtu=10218 mtu=9000
set [ find default-name=sfp-sfpplus2 ] comment="node3" l2mtu=10218 mtu=9000
set [ find default-name=sfp-sfpplus3 ] comment="node2" l2mtu=10218 mtu=9000
set [ find default-name=sfp-sfpplus4 ] comment="node1" l2mtu=10218 mtu=9000
/interface bridge
add comment="CRS3XX - vlan switch bridge" name=bridge-all-vlans pvid=404 vlan-filtering=yes
/interface vlan
add interface=bridge-all-vlans name=vlan404LabMgmt vlan-id=404
/interface bridge port
add bridge=bridge-all-vlans comment="node4 - NAS" interface=sfp-sfpplus1
add bridge=bridge-all-vlans comment="CRS3XX - mgmt" frame-types=admit-only-vlan-tagged ingress-filtering=yes interface=ether1 internal-path-cost=100 \
    path-cost=100
add bridge=bridge-all-vlans comment="node3" frame-types=admit-only-vlan-tagged interface=sfp-sfpplus2
add bridge=bridge-all-vlans comment="node2" interface=sfp-sfpplus3
add bridge=bridge-all-vlans comment="node1" frame-types=admit-only-vlan-tagged interface=sfp-sfpplus4
/ip neighbor discovery-settings
set discover-interface-list=!none
/interface bridge vlan
add bridge=bridge-all-vlans comment="bridge vlan404 LabMgmt - transit for MGMT IPAddress of the switch" tagged=bridge-all-vlans,ether1 vlan-ids=404
add bridge=bridge-all-vlans comment=SMB0 tagged=bridge-all-vlans,sfp-sfpplus2,sfp-sfpplus4 untagged=sfp-sfpplus1,sfp-sfpplus3 vlan-ids=8
add bridge=bridge-all-vlans comment=SMB1 disabled=yes tagged=sfp-sfpplus1,sfp-sfpplus2,sfp-sfpplus3,sfp-sfpplus4 vlan-ids=9
add bridge=bridge-all-vlans comment=iSCSI0 disabled=yes tagged=sfp-sfpplus1,sfp-sfpplus2,sfp-sfpplus3,sfp-sfpplus4 vlan-ids=10
add bridge=bridge-all-vlans comment=iSCSI1 disabled=yes tagged=sfp-sfpplus1,sfp-sfpplus2,sfp-sfpplus3,sfp-sfpplus4 vlan-ids=11
add bridge=bridge-all-vlans comment=GUEST-LAN disabled=yes tagged=ether1,sfp-sfpplus1,sfp-sfpplus2,sfp-sfpplus3,sfp-sfpplus4 vlan-ids=13
add bridge=bridge-all-vlans comment=VM-LAN disabled=yes tagged=ether1,sfp-sfpplus1,sfp-sfpplus2,sfp-sfpplus3,sfp-sfpplus4 vlan-ids=15
/ip dhcp-client
add disabled=no interface=vlan404LabMgmt
/ip dns
set servers=(FIRST-DNS-SERVER),(SECOND-DNS-SERVER)
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
set time-zone-name=Europe/Warsaw
/system identity
set name=CRS3XX
/system leds
set 0 disabled=yes
set 2 disabled=yes
set 4 disabled=yes
set 6 disabled=yes
set 8 disabled=yes
set 10 disabled=yes
set 12 disabled=yes
set 14 disabled=yes
/system ntp client
set enabled=yes primary-ntp=(PRIMARY-NTP-SERVER) secondary-ntp=(SECONDARY-NTP-SERVER) server-dns-names=0.pl.pool.ntp.org,1.pl.pool.ntp.org
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
Last update:2022.04.13