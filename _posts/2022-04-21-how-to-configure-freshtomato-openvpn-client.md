---
layout: post
title: "How to configure FreshTomato as OpenVPN client"
permalink: "/how-to-configure-freshtomato-openvpn-client/"
subtitle: "Paired with FreshTomato it is based on TLS1.3, Mikrotik - TLS1.2"
cover-img: /assets/img/cover/img-cover-tunnel.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-tunnel.jpg
tags: [HomeLab ,Networking ,FreshTomato ,OpenVPN]
categories: [HomeLab ,Networking ,FreshTomato ,OpenVPN]
---
This example is divided into three pieces. First describes the configuration for the tunnel between two FreshTomato flashed devices, next two sections combines FreshTomato client and Mikrotik Server, Router OS 7 and Router OS 6. It bases on lastest available releases of the software at the time of writing this post, which is 2022.3, Router OS 6.48.6 and 7.3.1.<br>
If you decide to follow along, you will end up with working OpenVPN tunnel.

## Prerequisites
+ Device with OpenVPN server (in this scenario Mikrotik device is the VPN server) or Device compatible with FreshTomato firmware (RT-N18U is K26ARM, 64K-NOSMP), alternativelly it can be any Desktop Operating System (then the scenario will vary, as you may be using vnet to vnet or vnet to peer) with [OpenVPN](https://openvpn.net/download-open-vpn/) software installed
+ OpenVPN server configured. It was described within the blog posts [How to configure on Freshtomato OpenVPN Server](how-to-configure-freshtomato-openvpn-server) and [How to configure Mikrotik OpenVPN Server - ROS 7.X - UDP](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros7/)
+ [EasyRSA 3](https://github.com/OpenVPN/easy-rsa) (preferably available on one of the bare metal devices), bring great benefit for the whole configuration
+ Optionally: Space to generate the certificate - you can use your Mikrotik device which may play the OpenVPN server role. You can also use the linux device, with the openssl. Wrote a blog post [Howto to compile openssl from sources](https://makeitcloudy.pl/how-to-compile-open-ssl-from-sources/). It can be compiled/installed on your linux/windows device, or one of your network appliances. OpenSSL is being used to remove the password from the private key, never the less if it is the EasyRSA which is being used to create certs, the *nopass* parameter did it's trick, and there is no need to use OpenSSL anymore
+ In order for the tunnel to work properly, proper routes along with firewall rules needs to be set on both ends. On FreshTomato, the routes are specified within the custom configuration
+ If the tunnel is established between two FreshTomato devices, where the OpenVPN Client has an LTE stick and on it's WAN the 192.168.X.X network range is being used, rember about the OpenVPN client configuration called *NAT on tunnel*, without this option checked in the GUI, iptables entries should arise within the config, for the traffic to be succesfully passed between the both ends of the OpenVPN tunnel

## ToDo
+ check if the Mikrotik Router OS 6.X supports CBC ciphers only, or there is some possibility to make use of GCM or Chacha

## Background
+ The overall configuration by prism of the configuration caveats like Auth or Ciphers will vary depending from the OpenVPN software/version playing the OpenVPN server and client role
+ Does the FreshTomato flashed devices supports AES-NI ? It depends from the processor architectures, looks that the devices equipped with ARMv7, does **not** support it, that's why for the data ciphers it might be wise using something [lighter](https://thehftguy.com/2020/04/20/what-aes-ciphers-to-use-between-cbc-gcm-ccm-chacha-poly/) like POLY-1305. Lighter ciphers makes sense, if this is the throughput which is your main concern. If you pass the thin protocols over your OpenVPN tunnel, then there is enough compute power to digest AES-GCM, with sufficient end user experience.
+ ARMv8 added support for AES-NI, more about this can be read [here](https://cs140e.sergio.bz/docs/ARMv8-A-Programmer-Guide.pdf), page 89
+ some says that overfilling the NVRAM can brick your router
```shell
# here is the way how to check how much is left
nvram show | grep size
size: 34621 bytes (30915 left)
```

## Introduction
At this stage it is assumed that the certificates has been already exported form the OpenVPN server. The initial steps are the same regardless of the protocol which is being used as transportation layer. 

1. Prepare certificates.
+ Certificate Authority
+ Client
+ Client key
2. Export the certificates from the router which plays the OpenVPN server role.
3. Remove the PASSWORDCERTPASSPHRASE from the private key (Maybe there is way but I could not figure this out, so this was an option to make the OpenVPN tunnel work).
3. Copy the content of the cetificates to FreshTomato based router, which act as a VPN client. This can be done within the *VPN Tunneling -> OpenVPN Client -> Custom configuration* section 
4. Test the connection

## Caveats
+ FreshTomato OpenVPN Client configuration slihgtly differs depending from the OpenVPN Server is TCP or UDP based.
+ It also differs depending from the fact if this is Mikrotik of FreshTomato acting as OpenVPN Server (different ciphers, auth's)
+ With OpenVPN there is no one fit's all approach, there are just too many versions and options, especially if one is staring from zero to hero

## Before you begin
Correct me if I'm wrong, never the less up to my experience in order let the FreshTomato establish the OpenVPN tunnel, password needs to be remove from the private key. OpenSSL is the tool to make this happen. The [OpenSSL section](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros6/) of the blog post describes how to get it.<br>
There is following assumption that the openssl.exe directory is already within the environmental paths so it is possible to execute it from the directory where the actual key is located.
```shell
# USERNAME - is the username which can be aligned with the PPP section on the Mikrotik OpenVPN server
# CN - is the name of the mikrotik device where the key has been generated
# PASSWORDCERTPASSPHRASE - is the password which was used during the openVPN Server configuration. Preferably it should
# differs between the user logins
# remove the [$USERNAME] and [$CN] and replace those with the real name of the from which you'd like to remove the password protection

# open your terminal or (powershell/cmd) and execute the following command
openssl rsa -passin pass:[$PASSWORDCERTPASSPHRASE] -in cert_export_[$USERNAME]$[$CN].key -out cert_export_[$USERNAME]$[$CN]_NoPass.key
```

Another aspect is the time which needs to be in sync between the OpenVPN server and client.

## OpenVPN Server is FreshTomato 2022.3 - UDP based tunnel - TLS 1.3
The WAN interface is the LTE stick, which is assigned with 192.168.8.X ip address.<br>
There is a chance that I was misled, never the less based on the observation unless within the *VPN Tunneling->OpenVPN Client->Basic tab* the Create NAT on tunnel option is checked, added routes were not enough for succesfull communication between the endpoints.<br>
Routers within the point to point connection could see each other, tunnel were established but that's all.

### FreshTomato OpenVPN Client - Basic Tab
This section is aligned with the OpenVPN server configuration described in abovementioned blogpost for FreshTomato OpenVPN Server. Both sides needs to fit within each other.
```shell
Start with WAN                   - checked
Interface Type                   - TUN
Protocol                         - UDP
Server Address                   - OVPNSERVERFQDN
Server Port                      - OVPNSERVERPORT
Firewall                         - Automatic
Create NAT on tunnel             - checked
Inbound firewall                 - unchecked
Authorization mode               - TLS
TLS control channel security     - Encrypt Channel V2
UserName/Password Authentication - checked
# UserName and Password should be aligned with the settings set on FreshTomato OpenVPN server, Advanced Tab
# where those values were defined inside the GUI, right after Allow User/Pass auth has been checked
UserName                         - USERNAME
Password                         - PASSWORDUSERLOGIN
UserName Authen. Only            - unchecked
Auth Digest                      - SHA512
```

### FreshTomato OpenVPN Client - Advanced Tab

```shell
Poll Interval                    - 0
# actually at this stage no entries are witin the routing policy, so it also can be set to No
Redirect Internet traffic        - Routing Policy
Accept DNS configuration         - Disabled
# VPN from 2.4 upwards picks the first protocol which is compatible on both ends
# that's why all those choices can be here, as the CHACHA is on the FreshTomato OpenVPN Server
# this is the chosen one during the negiotiation phase
Data ciphers                     - CHACHA20-POLY1305:AES-256-GCM:AES-256-CBC
Compression                      - Disabled
TLS Renegotiation Time           - -1
Connection retry                 - 30
Verify Certificate
(remote-cert-tls server)         - checked
# in this example the Common Name within the CA cert was: FreshTomato-ovpnServer
Verify Server Certificate Name
(verify-x509-name)               - Common name
```

### FreshTomato OpenVPN Client - Advanced Tab - Custom Configuration
There is no need to pass any extra parameters here, as all of them apart from the ones mentioned below has been already defined within the configuration. To let the tunnel communicate and set the verbosity level put following parameters.
```shell
verb 4
mute 10

route 172.16.88.0 255.255.255.0 vpn_gateway
```

This sums up the configuration for the OpenVPN tunnel between two FreshTomato devices.

## OpenVPN Server is Mikrotik ROS 7.X - UDP based tunnel
Another blog post [How to configure Mikrotik OpenVPN server on Router OS 7.X](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros7/) explains how to set it up, where this section specify the Freshtomato OpenVPN client configuration.<br>
FreshTomato 2022.3 cooperates with OpenVPN server interface restricted to tls-version=1.2 only.

### FreshTomato OpenVPN Client - Basic Tab
This section is aligned with the OpenVPN server configuration described in abovementioned blogpost for ROS 7.X. Both sides needs to fit within each other.

```shell
Start with WAN                   - checked
Interface Type                   - TUN
Protocol                         - UDP
Server Address                   - OVPNSERVERFQDN
Server Port                      - OVPNSERVERPORT
Firewall                         - Automatic
Create NAT on tunnel             - unchecked
Inbound firewall                 - unchecked
Authorization mode               - TLS
TLS control channel security     - Outgoing Auth(1)
UserName/Password Authentication - checked
# UserName and Password should be aligned with the PPP Secret parameters on the OpenVPN server
UserName                         - USERNAME
Password                         - PASSWORDUSERLOGIN
UserName Authen. Only            - unchecked
Auth Digest                      - SHA512
```

### FreshTomato OpenVPN Client - Advanced Tab
It suplements the Basic settings.

```shell
Poll Interval                    - 0
Redirect Internet traffic        - Routing Policy
Accept DNS configuration         - Disabled
Data ciphers                     - AES-256-CBC
Compression                      - Disabled
TLS Renegotiation Time           - -1
Connection retry                 - 30
Verify Certificate
(remote-cert-tls server)         - checked
# in this example the Common Name within the CA cert was: Mikrotik
Verify Server Certificate Name
(verify-x509-name)               - Common name
```

### FreshTomato OpenVPN Client - Advanced Tab - Custom Configuration
Let's repeat it again, similarly as it was for ROS 6.X, the password has been already removed from the private key and within the configuration the _NoPass.key content is being used, which is pasted between the *key* of the custom configuration section.
```shell
# the CN parameter which is used within the certicates and key section is the name of the Mikrotik Device
# which is being used to generate the certificates for the OpenVPN client
client
dev tun
proto udp
# remove the [OVPNSERVERFQDN] and replace it with the fully qualified domain name of the device which acts OpenVPN server
remote [OVPNSERVERFQDN]
# remove the [OVPNSERVERPORT] and replace it with a number which repsesents the port on which OpenVPN server is listening
port [OVPNSERVERPORT]
nobind
persist-key
persist-tun
tls-client
remote-cert-tls server
# cipher AES-256-CBC needs to go hand in hand with the available ciphers on the OpenVPN server interface
cipher AES-256-CBC
# auth parameter should be aligned with the auth on the OpenVPN server interface
auth SHA512
auth-nocache
resolv-retry infinite
verb 4
mute 10

# here you can add any extra routes towards the networks which are behind the OpenVPN Server
# [network] [netmask] [IP address of openVPN server interface]
# 10.0.6.254 is the address of the Mikrotik OpenVPN server interface
route 192.168.88.0 255.255.255.0 10.0.6.254
# here you can add any extra routes towards the networks which are behind the OpenVPN Server

<ca>
here you put the content of the client certificate - cert_export_[$USERNAME]@[$CN].cert
you put it all together within BEGIN and END certificate sections
</ca>

<cert>
here you put the content of the client certificate - cert_export_[$USERNAME]@[$CN].cert
you put it all together within BEGIN and END certificate sections
</cert>

<key>
here you put the content of the private key - cert_export_[$USERNAME]$[$CN]_NoPass.key
you put it all together within BEGIN and END certificate sections
</key>
```

This concludes the FreshTomato configuration, for the UDP tunnel towards RouterOS 7.X.
It looks that every time you chance something within your configuration, the tunnel is dropped and established from scratch.

## OpenVPN Server is Mikrotik ROS 6.X - TCP based tunnel
Another blog post [How to configure Mikrotik OpenVPN server on RouterOS 6.X](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros6/) explains how to set it up, where this section specify the Freshtomato OpenVPN client configuration.

### FreshTomato OpenVPN Client - Basic Tab
This section is aligned with the OpenVPN server configuration described in abovementioned blogpost. Both sides needs to fit within each other.

```shell
Start with WAN                   - checked
Interface Type                   - TUN
Protocol                         - TCP
Server Address                   - OVPNSERVERFQDN
Server Port                      - OVPNSERVERPORT
Firewall                         - Automatic
Create NAT on tunnel             - unchecked
Inbound firewall                 - unchecked
Authorization mode               - TLS
TLS control channel security     - Outgoing Auth(1)
UserName/Password Authentication - checked
# UserName and Password should be aligned with the PPP Secret parameters on the OpenVPN server
UserName                         - USERNAME
Password                         - PASSWORDUSERLOGIN
UserName Authen. Only            - unchecked
# SHA1 is being used for the compatibility with RouterOS 6
Auth Digest                      - SHA1
```

### FreshTomato OpenVPN Client - Advanced Tab
It's suplementary for the Basic settings.

```shell
Poll Interval                    - 0
Redirect Internet traffic        - Routing Policy
Accept DNS configuration         - Disabled
# Data ciphers parameter may vary depending what is your choice for the OpenVPN server
# It seems that Mikrotik ROS 6.X supports CBC only, even though the FreshTomato
# gives the possibility to use GCM or CHACHA-POLY1305 Ciphers
# this is why the stronges CBC has been used here
Data ciphers                     - AES-256-CBC
Compression                      - Disabled
TLS Renegotiation Time           - -1
Connection retry                 - 30
Verify Certificate
(remote-cert-tls server)         - checked
# the Common Name of the OpenVPN server certificate
Verify Server Certificate Name
(verify-x509-name)               - Common name
```

### FreshTomato OpenVPN Client - Advanced Tab - Custom Configuration
It is important to emphasis here, that as for the private key, the password has been already removed from it and within the configuration the _NoPass.key content is being used, pasted between the */key* section of the custom configuration.

```shell
# the CN parameter which is used within the certicates and key section is the name of the Mikrotik Device
# which is being used to generate the certificates for the OpenVPN client
client
dev tun
proto tcp
# remove the [OVPNSERVERFQDN] and replace it with the fully qualified domain name of the device which acts OpenVPN server
remote [OVPNSERVERFQDN]
# remove the [OVPNSERVERPORT] and replace it with a number which repsesents the port on which OpenVPN server is listening
port [OVPNSERVERPORT]
nobind
persist-key
persist-tun
tls-client
remote-cert-tls server
# cipher AES-256-CBC needs to go hand in hand with the available ciphers on the OpenVPN server interface
cipher AES-256-CBC
# auth SHA1 should be aligned with the auth on the OpenVPN server interface
auth SHA1
auth-nocache
resolv-retry infinite
# verb represents the verbosity which is a natural number between 0-11
verb 4
mute 10

# here you can add any extra routes towards the networks which are behind the OpenVPN Server
# [network] [netmask] [IP address of openVPN server interface]
# 10.0.6.254 is the address of the Mikrotik OpenVPN server interface
route 192.168.88.0 255.255.255.0 10.0.6.254

<ca>
here you put the content of the client certificate - cert_export_[$USERNAME]@[$CN].cert
you put it all together within BEGIN and END certificate sections
</ca>

<cert>
here you put the content of the client certificate - cert_export_[$USERNAME]@[$CN].cert
you put it all together within BEGIN and END certificate sections
</cert>

<key>
here you put the content of the private key - cert_export_[$USERNAME]$[$CN]_NoPass.key
you put it all together within BEGIN and END certificate sections
</key>
```

This concludes the Freshtomato configuration, for the TCP tunnel towards RouterOS 6.X.

## Links
+ OpenVPN [community resources](https://openvpn.net/community-resources/how-to/)
+ FreshTomato [Hardware Compatibility](https://wiki.freshtomato.org/doku.php/hardware_compatibility)
+ FreshTomato [Changelog](https://bitbucket.org/pedro311/freshtomato-arm/src/arm-master/CHANGELOG)
+ As of today, latest FreshTomato firmware can be downloaded from [freshtomato.org](https://freshtomato.org/downloads/freshtomato-arm/2022/2022.3/K26ARM/)

## Summary
That's it.<br>
If everything went well *VPN tunneling -> OpenVPN client -> Client1 -> Status* you should see natural numbers within all five columns TUN/TAP, TCP/UDP, Auth read bytes. This should proof that the packets are moving back and forth via the tunnel.<br>
Tested on FreshTomato 2022.3 K26ARM USB VPN-64K-NOSMP. Mikrotik ROS 6.48.6 and 7.3.1.<br>
Last update: 2022.07.01