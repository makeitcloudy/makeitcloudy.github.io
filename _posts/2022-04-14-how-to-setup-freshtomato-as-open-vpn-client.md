---
layout: post
title: "How to setup freshtomato as open vpn client"
permalink: "/how-to-setup-freshtomato-as-open-vpn-client/"
subtitle: "Setup your freshTomato router as OpenVPN client"
cover-img: /assets/img/how-to-setup-freshtomato-as-open-vpn-client/img-cover.jpg
thumbnail-img: /assets/img/how-to-setup-freshtomato-as-open-vpn-client/img-thumb.jpg
share-img: /assets/img/how-to-setup-freshtomato-as-open-vpn-client/img-cover.jpg
tags: [HomeLab ,Networking ,OpenVPN]
categories: [HomeLab ,Networking ,OpenVPN]
---
*As of 2022.04.14 - it is still in draft*

## Prerequisites
* OpenVPN server (in this scenario Mikrotik device is the VPN server)
* place where to generate the certificate - you can use your Mikrotik device which may play the OpenVPN server role
* openssl library compiled/installed on your linux/windows device, or one of your network appliances
* router flashed with alternative firmware (in this scenario freshTomato is being used), alternativelly it can be any Desktop Operating System (then the scenario will vary, as you may be using vnet to vnet or vnet to peer)

## Howto
1. Prepare certificates
2. Copy certificates to the router/OpenVPN which play the client role
3. Test the connection

## Summary
That's it.

Last update: 2022.04.14