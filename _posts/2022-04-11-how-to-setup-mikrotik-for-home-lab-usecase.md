---
layout: post
title: "How to setup mikrotik for home lab usecase"
permalink: "/how-to-setup-mikrotik-for-home-lab-usecase/"
subtitle: "There are plenty of alternative ways reaching this, but still mikrotik it's cost effective"
cover-img: /assets/img/how-to-setup-mikrotik-for-home-lab-usecase/img-cover.jpg
thumbnail-img: /assets/img/how-to-setup-mikrotik-for-home-lab-usecase/img-thumb.jpg
share-img: /assets/img/how-to-setup-mikrotik-for-home-lab-usecase/img-cover.jpg
tags: [HomeLab ,Networking]
categories: [HomeLab ,Networking]
---
*As of 2022.05.26 - it is draft due to the fact that there are just too many ways of doing it, which differs per device line RB vs CRS vs L2 and L3 approaches*

Your homelab can be powered from the network angle by different vendors, never the less for the price Mikrotik brings buch of functionalities which is worth the money. The interface may push you away a bit a first glimpse, but still it's worth investing a bit of time for the learning curve, especially being equipped with structualized content comming the TheNetworkBerg.

## Advise
1. Sit down, bring some calm and relax. Then perform the study over the Mikrotik documentation in context of the hardware which you have, and understand what are the differences between them from the prism of switch chip features being built in within the device. In simple words - each series of the devices, depending from the pricing range have different set of features. Rule is simple - the more you pay, more you get.
```bash
/interface ethernet switch print 
```
2. Adapt the vlan configuration to the switch chip and the product you have if you are interested in vlan switching speed close to the wire speed, being supported by ASIC and release the CPU of your router board to perform other activities. Material on youtube is great, but still the vendor documentation is the way to go, to adapt the scenario for the hardware which is in your disposal.

3. Backup your configuration, or use the winbox Safe Mode option.
4. Get use to the terminal.

## Background
What helped me much in the past, was the fact that for routers or switches it really does not matter whether it operates on the physical interfaces which you can see on the device when you are looking on it, or virtual interfaces which you can logically create within the device itself. So you can have physical ethernet ports, and abstraction layer where you put your virtual interfaces and then bind them with one of those physical interfaces, so it has a chance to reach the real wire.

## Prerequisities
You need the following:
+ it will be beneficial if you have some networking backgorund
+ need to spend some money on physical appliance, especially for the homelab usecase it is beneficial to have equippement which is not virutalized, it will pay off (ofcourse Eve-NG or GNS can also work)
## Where to find knowledge
There in incredible youtube channel worth donating created by [The Network Berg](https://www.youtube.com/c/TheNetworkBerg). 
He provides free MCTNA and more advanced trainings, bringing such level of detail which is enough for configuring your network device up to this extend that it can easily support your virtualized infrastructure or provide the service for your SOHO (small office home office) deployment.
The fact is that for the home lab, you do not need that great amount of routing, unless you are configuring a specific usecase, never the less for simple scenarios, products like RB951 series should be more than enough.

+ Mikrotik products can be found [here](https://mikrotik.com/products/)
+ Mikrotik free MCTNA training provided by TheNetworkBerg can be found [here](https://www.youtube.com/playlist?list=PLJ7SGFemsLl3XQhO8g0hHCrKnC6J3KURk). Please toss him some coin, or become a Patron.
+ CCR series will definitelly bring great benefit for the overall configuration, never the less combination of RB and a CRS switch which will act as a L2 device will also suffice up to this extend that some interesting architectures can be built on top of that.
## Relation between virtual elements which builds the configuration
# Software defined vlans (router scenario)
1. VLANs relate to Interfaces - you can bind VLAN to interfaces for instance: (ether2-ether5) - once this is done on two sides of the wire, the devices (if you have not disabled this) can see each other as a neighbours on L2 level.
2. IP addresses relate to VLANs - you assign an IP address to VLAN, once this is done on two sides of the wire and some traffic is generated then the devices can see each other as a neighbours on L3 level.
# Bridged vlans (switch scenario)
1. Introduce a bridge to the network
2. For each vlan create a bridge which will be linked to that vlan
3. Assign the vlan to the bridge. At this stage VLAN it tagged on ether port, but it is also bridged with our bridge vlan. Then assign port to the bridge (this steps combines the physical port with the virutal entity which is the bridge) creating the access mode scenario. 
Ether port will transport untagged VLAN, where the vlan interface is tagged and going to the uplink / trunk.

## Rules
0. Maybe I have not grasp it well, never the less my observation is that sometimes Mikrotik allows you performing the configuration which seems to be working for you, but causes spikes on the CPU causing the device a bit of sluggish. When you understand what you are doing and you follow the recommendation there is lesser chance to step into this, never the less you may reach the blind alley.
```bash
# it will show you which daemon may bring high utilization of your device
/tool profile duration=60s
```
0. VLAN configuration differs between the CRS3XX, CRS2XX/CRS1XX series and other devices like router boards. CRS3XX series support vlan filtering on the switch chip, where cheaper devices like RB951-2n, does not. All details about those aspects can be found here for [CRS3XX series](https://wiki.mikrotik.com/wiki/Manual:CRS3xx_series_switches#Features) and [another series](https://wiki.mikrotik.com/wiki/Manual:Interface/Bridge#Bridge_Hardware_Offloading.
Basic vlan settings configuration can be found [here](https://wiki.mikrotik.com/wiki/Manual:Basic_VLAN_switching) and those become really straightforward and almost self explanatory, once you have some previous hands on experience and had failed misserably trying to understand all different concepts of achieving similar functionality. In fact there are just too many ways for doing this, at least for someone whose core skill is not networking.
1. Configure bridges (vlans on bridges and assign ethernet ports to bridges) where you like your router board to act as a switch, in such case, do not configure the interfaces (apart from so called virtual interface for the CPU, so you can manage your device via IP type of connectivity, which is handy when the device does not have a console port) - this applies to Router Boards as well as CRS (your CRS will act as a switch, with the benefit of giving you the capability of managing it via winbox) and CCR's. Then add the ports to bridge.
```bash
/interface bridge port> print
Flags: X - disabled, I - inactive, D - dynamic, H - hw-offload 
 #     INTERFACE                                        BRIDGE                                       HW  PVID PRIORITY  PATH-COST INTERNAL-PATH-COST    HORIZON
 0 I H ether2                                           bridge-all-vlan                              yes 1342     0x80         10                 10       none
 1 I H ether3                                           bridge-all-vlan                              yes 1343     0x80         10                 10       none
 2 I H ether4                                           bridge-all-vlan                              yes 1344     0x80         10                 10       none
 3   H ether5                                           bridge-all-vlan                              yes    1     0x80         10                 10       none
 4 XI   wlan1                                            bridge-all-vlan                                     1     0x80         10                 10       none
 5 I   ether1                                           bridge-all-vlan                              yes   70     0x80         10                 10       none
```
2. Configure interfaces (vlans on interfaces) when you like your router board to act as a L3 device. Regardless of L3 or L2, assign the vlan to the bridge, not ethernet ports.
```bash
/interface vlan
add interface=bridge-all-vlans name=vlan70 vlan-id=70
```
3. Regardless of the desired result L2 or L3, assign your Interface level VLAN's to bridge (remember to stick with one bridge, for making use of the switch chip with hardware offloading - do not create more than one bridge). Do not configure anything VLAN related on the switch level.
4. Bridge level vlans (PVID is assigned on the bridge port level, tag is being set on the bridge vlan)
```bash
xxx
```
5. If on your device uplink interface, someone gives you a tagged frame, set the corresponding bridge VLAN ID's on the ethernet port which is your uplink, as well as on the bridge
```bash
/interface bridge vlan
add bridge=bridge-all-vlans tagged=bridge-all-vlans,ether1 vlan-ids=70
```
6. Then configure the DHCP client and bound it to the vlan interface (this will guarantee that you will get the IP address on the uplink interface on your device). Same rule applies for the DHCP server, assign it to the VLAN interface, not bridge, neither the ether port.
```bash
/ip dhcp-client
add comment=uplink disabled=no interface=vlan70 use-peer-dns=no use-peer-ntp=no
```
7. Always configure your Address Lists with the netmask suffix otherwise you may encounter problems with communication. The communication between devices immediatelly started to occur when at the end of the address you have /subnet like /24 or /29 or whatever else.
```bash
add address=10.0.70.254/24 interface=vlan70 network=10.0.70.0
```
8. Set the VLAN Filtering on the bridge level (you have single bridge only) at the end of your configuration of the tagged and untagged settings on the Bridge level Vlans. Once this is set you'll immediatelly see the effect VLAN's bridge GUI section of the winbox showing within the intrface which are tagged and which are not.

15. Configure the IP Services List at the very end of your configuration when the addressing scheme is already set up, and won't change during the configuration of the device (this will lower the chance that you'll cut off yourself permanently - this applies for the conditions where you do not have physical access to the device, and there is no console port) - set ssh and set winbox sections
```bash
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set ssh address=10.0.45.0/24,192.168.45.0/24
set api disabled=yes
set winbox address=10.0.45.0/24,192.168.45.0/24
set api-ssl disabled=yes
```
## Howto
+ Backup your current configuration first, save it on the device, and copy it somewhere externally, so you have an easy way to revert to previous configuration, which hopefully have been already working well for you. I assure you that if you are not touching those devices in regular basis, you will forget how to do it by covering all different angles and meandres of the setup.
+ Be prepared for the fact that when you will be setting the things up, you will cut yourself of from the device at least few times, so apply the steps of the configuration this way that you can still connect to the device without the need to reset the configuration. What I mean by that is the following, apply the desired configuration on one logical interface, and then once it is confirmed it works iterate further. Another option is to use the Safe Mode within the winbox gui.
+ How to [reset your configuration to factory defaults](https://wiki.mikrotik.com/wiki/Manual:Reset) (it is helpfull when you cut yourself off, have not been using Safe Mode and yo have a device which does not have a COM port, or do not remember the password for your device anymore). Reset to factory defaults with making use of the physical button or within the command from the terminal (after 5s the ACT icon should start blinking), does not remove the backup of your configuration from the filesystem, neither any other files like the ones used for setting up the VPN.
+ Once you did that, your device will use the addresses from the 192.168.88.X range, and you can login with the IP address of the default gateway which for instance for RB951-2n (which I'm using for different use case than the heart of the network for the lab, is 192.168.88.1).
+ Small and cheap devices like RB951-2n is a perfect box for a trial and error, it's handy, reliable, fast enough for most usecases to learn Router OS version 6.
+ You take the winbox and log into the device with the use of empty password, and login: admin. won't have to start from scratch in case your plan is to perform a small adjustment within the configuration. 
+ Once the device is reset to factory defaults, and you won't agree during it's boot for applying of the auto generated script the only option to connect there is still with the use of winbox, but one layer lower with use of neighbours discovery, and it's MAC address instead of the IP address, until you configure one on your bridge or VLAN.

There are three ways of configuring VLANs
1. [software defined VLANs](https://www.youtube.com/watch?v=4BOYqtV4MCY&list=PLJ7SGFemsLl1QUNkgAbGj9ldlWRrr8zMj&index=1&t=501s) (let's call it old fashioned way) - here vlans are bound to the interfaces
2. [bridged VLANs](https://www.youtube.com/watch?v=4BOYqtV4MCY&list=PLJ7SGFemsLl1QUNkgAbGj9ldlWRrr8zMj&index=1&t=848s) (let's call it modern way) - here vlans are bound to the bridges
+ for each VLAN there is a separate bridge linked to that VLAN
3. [switch chip powered VLANs](https://www.youtube.com/watch?v=4BOYqtV4MCY&list=PLJ7SGFemsLl1QUNkgAbGj9ldlWRrr8zMj&index=1&t=1168s) (those are the VLANs which are powered by the ASIC not the CPU, so the Central Processing Unit of your device, has some more room left for processing other type of activies like checking the firewall rules, which you'll put on top of your VLAN configurations). There is one remark - your device needs to have [switch chip](https://wiki.mikrotik.com/wiki/Manual:Switch_Chip_Features), in our example RB951-2n has one called Atheros7240, which serves the ethernet ports 2-5, so the ethernet port 1 can be used as the uplink, and what's more can be powered over PoE. Brilliant isn't ?
3.1 Only one bridge on your device. Stick with this rule, as from what I recall from different lecture if there are more bridges the Hardware Offloading have issues with itself, and the performance is degradaded.
3.2 Call the bridge bridge-switch-chip
3.3 Add ether port to the bridge, and set the PVID of the VLAN on the port in the BRIDGE context
3.4 At this stage VLAN tab in the bridge context should not show anything
3.5 Still in the BRIDGE context go the the bridge tab, into the VLAN tab of the bridge-switch-chip and enable vlan filtering (just enable it, without performing any furhter configuration). At this stage it is creating the dynamic interfaces on the bridge.
3.6 At this stage VLAN tab in the BRIDGE context should show the VLANs which are created on the interfaces along with the detail, whether those are tagged or untagged. PS. when you configure the VLANs on the interfaces which are just logically configured, and nothing is configured to them, what you will see in this tab will be the VLAN with ID 1 (provided the VLAN1 has been added to the bridge)
3.7 At this point you can copy the dynamically configured bridge vlans, and decide on which ethernet ports those should tagged and untagged

3.100 Vlan Filtering (do it last)

## Useful commands
+ When something goes wrong, or you'd like to take a fresh start
```bash
/system reset-configuration
```
+ Router Boards, then put themselves into 192.168.88.X network range, and once you are connected to one of it's ports (preferably the ones which are served by the switch chip, so you can use the other one for your uplink port)
## Links
+ [OepnSSL build on Windows](https://michlstechblog.info/blog/openssl-build-on-windows/)
+ [OpenVPN Server and certificate management on MikroTik](https://gist.github.com/SmartFinn/8324a55a2020c56b267b)
+ [Enable OpenVPN Server on Mikrotik RouterOS](https://lunar.computer/posts/mikrotik-routeros-openvpn/)
## Summary
That's it.

Last edit: 2022.04.11
