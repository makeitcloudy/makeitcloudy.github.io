---
layout: post
title: "FreshTomato versus voip phone"
permalink: "/freshtomato-vs-voip-phone/"
subtitle: "A bit of a hussle, never less made it work"
cover-img: /assets/img/freshtomato-vs-voip-phone/img-cover.jpg
thumbnail-img: /assets/img/freshtomato-vs-voip-phone/img-thumb.jpg
share-img: /assets/img/freshtomato-vs-voip-phone/img-cover.jpg
tags: [Networking ,FreshTomato ,VoIP]
categories: [Networking ,FreshTomato ,VoIP]
---
Even though it is not very popular in 2022, some still make use of VoIP phones, which are reliable as tanks unless configured properly. Even though previously it was working flawlessly, this time it cost me a bit of time to make it work.

## Background
There is ton of contradictory information in the internet, some of those are comming from the year range of 2009 - 2013, it seems that the support for the VoIP device has ended on 2013. But still the ergonomy of the device is really good to be frank, and once it's configured up to this extend that it is operational it just works. There are also VoIP providers, which can still offer you a landline number from your zone, or you can setup a PBX by your own, for example Asterisk or [FreePBX](https://nerdonthestreet.com/episode/tech/freepbx-showcase) based - there is a great video about it, which has been prepared by the NerdOnTheStreet.

The VoIP device interface, does not differ much between each other, if you have a spare analog phone laying somewhere and would like to give it a spin and second life, provided it send the digits with DTMF. If you like go the true geek way, there are also transceivers which can transpond the analog signals (pre DTMF - can not recall the acronym at the moment) to DTMF, and you can plug the phone from early 50ties of XX century to the VoIP gateway via the FXS port.

## Prerequisites
+ FreshTomato (in my case it was 2021.8)
+ VoIP phone (Cisco SPA502G, firmware 7.5.2 - there is 7.5.5 available, it was even uploaded as per the firmware update software to the device, never the less for some reason it could not get flashed. Remeber to set the option 'Upgrade Enable' to yes, to make it even possible). As an alternative you can also use the Grandstream or Cisco SPA gateway.

## Howto
+ Configure your router, that it assign an IP address and DNS servers to the phone
+ **SIP-ALG** - here the confusion comes, plenty threads are suggesting that you need to disable this, by unchecking it within the interface of your router in the Advanced tab -> Contrack/NetFilter -> Tracking/NAT Helpers - uncheck SIP.<br>
On Mikrotik it indeed do it's trick and the phone registers to the VoIP provider, as well as it immediatelly allows incoming calls. The overall experience if seamless.<br>
```bash
# regardless of the mikrotik series, the SIP-ALG once disabled, causes the your VoIP phone works flawlessly
[user@rb951-2n] > /ip firewall service-port export 
/ip firewall service-port
set sip sip-direct-media=no
```

+ Maybe I need more testing, never the less on FreshTomato (2021.8 release), for some reason (uknown to me), I was not very succesfull with combination of Cisco SPA502G. The overall experience was not that seamless as it was on Mikrotik.
+ Even though users are reporting (uncheking SIP) did the trick for them it did not for me. **That's why on FreshTomato, I left it checked**. All other options FTP, GRE/PPTP, H.323, RTSP are unchecked.
+ Port forwarding - not configured. Even though I have been trying with the 5060, 16384-16583 (some says that RTP requires two ports being open for one device to pass the connection stream)
```bash
#EXT tab
* VoIP device, configured as per your VoIP provider suggestions
* NAT Settings on the VoIP device - 
1. NAT Keep Alive: Enable (here I am not 100% sure whether it really have to be set, and some says that it cause the SPA502G to hung - will observe that)
2. NAT Keep Alive Dest: $PROXY
3. SIP Transport: UDP
4. SIP Port: 5060
5. Use Outbound proxy: Yes
6. Use OB Proxy In Dialog: Yes
7. Register: Yes
8. Use DNS SRV: No
9. DNS SRV Auto Prefix: No
10. Proxy Fallback Interval: 900
```
+ The intresting thing is that with abovementioned configuration, when your FreshTomato is the router which serves you the Internet, then you connect your Mikrotik to it and to your Mikrotik router you plug the phone, it just works and the incomming calls are working just fine. The topology looks like this: INTERNET -> FRESHTOMATO -> MIKROTIK -> PHONE.

## Summary
This was tested with FreshTomato 2021.8, there is great chance it will also work with other releases of the firmware.<br>
Last update: 2022.04.14