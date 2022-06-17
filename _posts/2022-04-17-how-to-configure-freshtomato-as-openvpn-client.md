---
layout: post
title: "How to configure FreshTomato as OpenVPN client"
permalink: "/how-to-configure-freshtomato-as-openvpn-client/"
subtitle: "Configure your FreshTomato router as OpenVPN client"
cover-img: /assets/img/cover/img-cover-tunnel.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-tunnel.jpg
tags: [HomeLab ,Networking ,FreshTomato ,OpenVPN]
categories: [HomeLab ,Networking ,FreshTomato ,OpenVPN]
---
*As of 2022.04.16 - draft*

## Prerequisites
+ OpenVPN server (in this scenario Mikrotik device is the VPN server)
+ Space to generate the certificate - you can use your Mikrotik device which may play the OpenVPN server role. You can also use the linux device, with the openssl. Howto get the openssl compiled is shared [here](https://makeitcloudy.pl/how-to-compile-open-ssl-from-sources/).
+ OpenSSL library compiled/installed on your linux/windows device, or one of your network appliances
+ Preferably router flashed with alternative firmware (in this scenario FreshTomato is being used), alternativelly it can be any Desktop Operating System (then the scenario will vary, as you may be using vnet to vnet or vnet to peer) with [OpenVPN](https://openvpn.net/download-open-vpn/) software installed.

## Howto
1. Prepare certificates.
+ Certificate Authority
+ Client
+ Client key
2. Export the certificates from the router which plays the OpenVPN server role.
3. Copy the content of the cetificates to FreshTomato based router, which act as a VPN client.
3. Test the connection

## Summary
That's it.<br>
Tested on FreshTomato 2021.8 K26ARM USB VPN-64K-NOSMP. As of today, latest update for the firmware can be downloaded from [freshtomato.org](https://freshtomato.org/downloads/freshtomato-arm/2022/2022.3/K26ARM/)<br>
Last update: 2022.04.17