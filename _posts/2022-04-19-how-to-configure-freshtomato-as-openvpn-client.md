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
This example describes how to setup the TCP or UDP OpenVPN tunnel between your FreshTomato client and Mikrotik Server. It bases on lastest available releases of the software at the time of writing this post. If you decide to make use of it from the begging up to the end you will end up with working OpenVPN tunnel.

## Prerequisites
+ Device with OpenVPN server (in this scenario Mikrotik device is the VPN server)
+ Device compatible with FreshTomato firmware (RT-N18U is K26ARM, 64K-NOSMP)
+ Space to generate the certificate - you can use your Mikrotik device which may play the OpenVPN server role. You can also use the linux device, with the openssl. Wrote a blog post [Howto to compile openssl from sources](https://makeitcloudy.pl/how-to-compile-open-ssl-from-sources/).
+ OpenSSL library compiled/installed on your linux/windows device, or one of your network appliances
+ Preferably router flashed with alternative firmware (in this scenario FreshTomato is being used), alternativelly it can be any Desktop Operating System (then the scenario will vary, as you may be using vnet to vnet or vnet to peer) with [OpenVPN](https://openvpn.net/download-open-vpn/) software installed.
+ In order for the tunnel to work properly, proper routes along with firewall rules needs to be set on both ends. On FreshTomato, the routes are specified within the custom configuration.

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

FreshTomato configuration slihgtly differs depending from the OpenVPN Server is TCP or UDP based.

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

### OpenVPN Server is Mikrotik ROS 6.X - TCP based tunnel
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
Auth Digest                      - SHA1
```

### FreshTomato OpenVPN Client - Advanced Tab
It's suplementary for the Basic settings.

```shell
Poll Interval                    - 0
Redirect Internet traffic        - Routing Policy
Accept DNS configuration         - Disabled
Data ciphers                     - AES-256-CBC
Compression	                     - Disabled
TLS Renegotiation Time           - -1
Connection retry                 - 30
Verify Certificate
(remote-cert-tls server)         - checked
# in this example the Common Name within the CA cert was: mkt
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
verb 4
mute 10

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

This concludes the Freshtomato configuration, for the TCP tunnel towards RouterOS 6.X.

### OpenVPN Server is Mikrotik ROS 7.X - UDP based tunnel
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
It's suplementary for the Basic settings.

```shell
Poll Interval                    - 0
Redirect Internet traffic        - Routing Policy
Accept DNS configuration         - Disabled
Data ciphers                     - AES-256-CBC
Compression	                     - Disabled
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
# auth SHA1 should be aligned with the auth on the OpenVPN server interface
auth SHA512
auth-nocache
resolv-retry infinite
verb 4
mute 10

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

## Links
+ OpenVPN community [resources](https://openvpn.net/community-resources/how-to/)
+ FreshTomato [Hardware Compatibility](https://wiki.freshtomato.org/doku.php/hardware_compatibility)
+ FreshTomato [Changelog](https://bitbucket.org/pedro311/freshtomato-arm/src/arm-master/CHANGELOG)
+ As of today, latest FreshTomato firmware can be downloaded from [freshtomato.org](https://freshtomato.org/downloads/freshtomato-arm/2022/2022.3/K26ARM/)

## Summary
That's it.<br>
If everything went well *VPN tunneling -> OpenVPN client -> Client1 -> Status* you should see natural numbers within all five columns TUN/TAP, TCP/UDP, Auth read bytes. This should proof that the packets are moving back and forth via the tunnel.<br>
Tested on FreshTomato 2022.3 K26ARM USB VPN-64K-NOSMP. Mikrotik ROS 6.48.6 and 7.3.1.<br>
Last update: 2022.06.19