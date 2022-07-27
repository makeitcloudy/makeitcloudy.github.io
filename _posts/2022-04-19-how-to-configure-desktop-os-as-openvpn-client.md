---
layout: post
title: "How to configure Desktop OS as OpenVPN client"
permalink: "/how-to-configure-desktop-os-as-openvpn-client/"
subtitle: "OpenVPN tunnel between your desktop Operating system and OpenVPN server"
cover-img: /assets/img/cover/img-cover-tunnel.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-tunnel.jpg
tags: [HomeLab ,Networking ,OpenVPN]
categories: [HomeLab ,Networking ,OpenVPN]
---
 In this example the OpenVPN tunnel is set up between the Mikrotik OpenVPN server and the OpenVPN client installed on Linux or Windows Desktop Operating system. It's the protocol which should match, it does not matter on top of what compute layer it operates on unless both ends can talk it's language.

## Prerequisites
+ Desktop operating system (Linux / Windows )
+ OpenVPN Server configuration completed [TCP Router OS 6.X](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros6/) or [URP Router OS 7.X](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros7/)
+ Exported certificates from the OpenVPN configuration, already on your client
+ Private keys are password protected here, so during establishing the connection, the end user will have to put the PASSWORDCERTPASSPHRASE defined during the abovementioned configuration
+ Both operating systems can set the UDP or TCP tunnels, still in this blog post the UDP will be depicted on the example of linux, where TCP on windows.

## Links
+ OpenVPN [wiki](https://community.openvpn.net/openvpn/wiki/GettingStartedwithOVPN) - Getting Started with OpenVPN

## ToDo
+ modify the client configuration to make use of udp4 or tcp4-client
+ add the client configuration for FreshTomato based OpenVPN server

## Linux - UDP based client
1. Install OpenVPN client

```shell
## copy into the /etc/openvpn/client/
## Mikrotik.crt
## ovpn-Client1.crt
## ovpn-client1.key
nano secret
## secret contains the
## USERNAME
## PASSWORDUSERLOGIN
# .ovpn contains your OpenVPN client configuration
```

2. Edit .ovpn file

```shell
client
dev tun
## in this case the tunel is established towards the ROS 7.X UDP based
proto udp
## remote equals to the Fully Qualified Domain Name for the WAN interface of OpenVPN serve
remote [OpenVPN Server FQDN]
## port is the natural number aligned with the OpenVPN server configuration
port [OVPNSERVERPORT]
nobind
persiste-key
persist-tun
tls-client
remote-cert-tls server
ca /etc/openvpn/client/Mikrotik.crt
cert /etc/openvpn/client/ovpn-Client1.crt
key /etc/openvpn/client/ovpn-Client1.key
verb 4
mute 10
## auth and cipher should be aligned with the auth and cipher of openvpn-server interface
cipher AES-256-CBC
auth SHA512
auth-user-pass /etc/openvpn/client/secret
auth-nocache
## here comes the routes for the networks which should be accessible via the tunnel
route 192.168.88.0 255.255.255.0 10.0.6.254
## remember adding a correct route on the OpenVPN server towards the network of your
## OpenVPN client - the easiest way would be to check the network where your device
## is located and prepare the route upfornt the tunnel is established.
```

Proof that it can be established

```shell
sudo openvpn --config /etc/openvpn/client/ovpn-Client1.ovpn
ping 192.168.88.1
## you should get reply provided the firewall allows such traffic
```

## Windows - TCP based tunnel
1. Install [OpenVPN client](https://openvpn.net/community-downloads/)

```powershell
# here is the location where the configuration files are stored
Set-Location -Path $env:USERPROFILE\OpenVPN\config
Get-ChildItem

# you should customize the ovpn file and create the secret
# .ovpn contains your OpenVPN client configuration
# secret contains the
# USERNAME
# PASSWORDUSERLOGIN
# from the variable section of OpenVPN server configuration

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/19/2022   9:43 PM           1176 Mikrotik.crt
-a----         4/19/2022   9:43 PM           1147 ovpn-Client1.crt
-a----         4/19/2022   9:43 PM           1858 ovpn-Client1.key
-a----         4/19/2022   9:55 PM            481 ovpn-Client1.ovpn
-a----         4/19/2022  10:03 PM             27 secret
```
2. Once the certificates are put into abovementioned directory, go ahead and edit the .ovpn file

```shell
client
dev tun
## in this case the tunel is established towards the ROS 6.X TCP based
proto tcp-client
## remote equals to the Fully Qualified Domain Name for the WAN interface of OpenVPN serve
remote [OpenVPN Server FQDN]
## port is the natural number aligned with the OpenVPN server configuration
port [OVPNSERVERPORT]
nobind
persist-key
persist-tun
tls-client
remote-cert-tls server
ca ca.crt
cert ovpn-Client1.crt
key ovpn-Client1.key
verb 4
mute 10
cipher AES-256-CBC
auth SHA1
## the auth-user-pass parameter is the name of the file which contains the
## USERNAME and PASSWORDUSERLOGIN
auth-user-pass secret
auth-nocache
## here comes the routes for the networks which should be accessible via the tunnel
route 10.0.2.0 255.255.255.0 10.0.6.254
route 10.0.3.0 255.255.255.0 10.0.6.254
route 10.0.4.0 255.255.255.0 10.0.6.254
route 10.0.5.0 255.255.255.0 10.0.6.254
## remember adding a correct route on the OpenVPN server towards the network of your
## OpenVPN client - the easiest way would be to check the network where your device
## is located and prepare the route upfornt the tunnel is established.
```

The tunnel can be established without routes, but then there won't be any traffic between the device which are supposed to use the tunnel. Please remember about it, when you are not a network guy, it can be easily forgotten.

3. The OpenVPN GUI should be located next to the clock within the start menu, when you right click on it, and type the *PASSWORDCERTPASSPHRASE* for the private key of the certificate which is in use, then you should be successfully logged in.

```powershell
## this can be easily proofed by
ipconfig
## apart from your regular IP address you will also be shown with
Unknown adapter OpenVPN:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 10.0.6.253
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :
## there is no default gateway, as only the networks covered by routes are sent over the tunnel
```

## Summary
That's it.<br>
It was tested on Windows OS and Lubuntu 20.04.03 LTS.<br>
OpenSSL 1.1.1o, OpenVPN 2.5.6-I601, Router OS 6.48.6<br>
OpenSSL 1.1.1f, OpenVPN 2.4.7, Router OS 7.3.1<br>
Last update: 2022.05.19