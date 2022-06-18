---
layout: post
title: "How to configure mikrotik as OpenVPN client"
permalink: "/how-to-configure-mikrotik-as-openvpn-client/"
subtitle: "Not that difficult, when you failed few times already"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
categories: [HomeLab ,Networking ,Mikrotik, OpenVPN]
---
*2022.04.17 - Draft*
In this example the connection takes place between two Mikrotik devices, never the less it's protocol which should match, it does not matter on top of what compute layer it operates on. In this scenario the regular traffic is routed through the Internet, where the target networks defines within the routes, are traversed over the OpenVPN tunnel.
## Prerequisites
+ DNS alias or IP address of your OpenVPN server
+ Mikrotik RB/CCR device
+ OpenVPN Server configured

## Howto
The procedure contains few steps which should be executed in following order
1. import the files exported during the OpenVPN server configuration
2. import the certificates
3. adress lists will be created dynamically once the OpenVPN is established, no need to create it manually
4. create PPP profile
5. create PPP interface
6. add routes (dst is the target network you reach over the tunnel)

## Router OS 7.3.1
.

## Router OS 6.48.6
This section is dedicated for the Router OS version 6.
### Configuration - ROS 6.X - defining variables
Open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window.<br>
**script does not work if the passwords contains \ *backslash***

```shell
:global CN [/system identity get name]
:global USERNAME "Client1"
:global OVPNPROFILENAME "ovpn-profile"
:global OVPNIPADDRESS "10.0.6.254"
## 2022.04.21 - should it be 8character long
:global PASSWORDCERTPASSPHRASE "12345678"
:global PASSWORDUSERLOGIN "clientPassword"
:global OVPNCLIENTIPADDRESS "10.0.6.253"
```
### Configuration - ROS 6.X - Execute this piece of code on device which acts as OpenVPN client
```shell
## add a user
/ppp secret
add name=$USERNAME password="$PASSWORDUSERLOGIN" profile="$OVPNPROFILENAME" service=ovpn \
local-address="$OVPNIPADDRESS" remote-address="$OVPNCLIENTIPADDRESS"

## generate a client certificate
/certificate
add name=client-template-to-issue copy-from=client-template \
  common-name="$USERNAME@$CN"
sign client-template-to-issue ca="$CN" name="$USERNAME@$CN"
:delay 10

## export the certificate
/certificate
export-certificate "$CN" export-passphrase=""
export-certificate "$USERNAME@$CN" export-passphrase="$PASSWORDCERTPASSPHRASE"

## 
```
## Summary
I'm sure there are better ways doing it, but still it's a good starting point.<br>
It was tested on RB951G 7.3.1 and CCR with ROS 6.48.6<br>
Last update: 2022.06.18