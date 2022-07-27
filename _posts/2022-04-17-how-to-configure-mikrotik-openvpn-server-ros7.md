---
layout: post
title: "How to configure Mikrotik OpenVPN Server - ROS 7.X - UDP"
permalink: "/how-to-configure-mikrotik-openvpn-server-ros7/"
subtitle: "Router OS 7.3.1"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
categories: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
---
This is an updated version of previous blog post which was describing [how to configure OpenVPN server on ROS 6.X](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros6/), which brings the updates towards ROS 7.X. Please read this post before applying those settings, unless you can ammend current configuration accordingly to suit your needs.<br>
When you have a working OpenVPN on TCP, switching to UDP is like turning the Protocol switch from one to another, and modifying the firewall rules on the device acting as Mikrotik OpenVPN Server, and if it is the case on the router in front of your Mikrotik which may be performing another NAT, as OpenVPN (especially if you work with thin protocols, performs well enough to serve such scenarios).

## Prerequisites
+ DDNS configuration on top of your dynamic IP address or static IP address
+ Mikrotik Routerboard/CCR device
+ NTP server configured properly, so the time and date is in sync
+ OpenSSL (not a must)
+ A bit of Mikrotik knowledge

## Background
+ when your Mikrotik Router which plays the OpenVPN server role is behind nat, make sure the UDP port is forwarded accordingly, especially if you make a switch from previous releases
+ one of the easiest way whether there are any packets knocking to the port (OpenVPN service) is on firewall level by the amount of Bytes received. In case the port is not forwarded accordingly then it will show 0
+ TLS handshake failure at least happens when the Auth parameter on the OVPN Server Interface, does not go hand in hand between the client and the server or the port forwarding on the NAT is not configured propely or using wrong protcol TCP instead of UDP or vice versa

## Features availbale in ROS 7.X which was not exposed before
1. UDP procol (since 7.0beta3), UDP is being used by L2TP or Wireguard
2. IPv6
3. LZO compression
4. Authentication without username and password

## Auth and Ciphers
+ It seems that the AES-NI instructions are hardware accelerated (here I'm not completelly sure), but on an example of [IPSEC](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec) it looks that on some devices AES128,256 are, where 512 are not
+ I could not find the exact details within the documentation whether the AES-GCM is being use with ROS 7, or it is CBC - probably again it may vary on the device, but I'm not sure. (Hopefull debug log will reveal that detail, comparing the RouterBoards and CCR/CRS)

```shell
 #RB951G 7.3.1
 /system logging add topics=ovpn,!debug
 /log print
 #ovpn,info : using encoding - ......
 #when you finished disable ovpn logging
 /system logging add topics=ovpn,!debug disabled=yes
```

+ Some says that, OpenVPN is only a toy and the [IPSec](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec) is the way to go
+ Why even bother about few strings CBC/GCM is explained [here](https://alicegg.tech/2019/06/23/aes-cbc.html)


### Configuration - defining variables
Open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window. As per Mikrotik [documentation](https://help.mikrotik.com/docs/display/ROS/OpenVPN) password is limited to 233, where the username can have up to 27 characters.
**passwords can not contain \ *backslash***

```shell
:global CN [/system identity get name]
:global PORT 4911

:global DAYSVALIDCA 945
## it does not make sense for me that the SERVER is bigger/longer than CA which is used to sign the server cert
:global DAYSVALIDSERVER 945
:global DAYSVALIDCLIENT 365

:global OVPNCLIENTCERTIFICATETEMPLATENAME "ovpn-ClientTemplate"
:global OVPNCLIENTCERTIFICATETEMPLATECOMMONNAME "ovpnClientTemplate"
:global OVPNPROFILENAME "ovpn-profile"
:global OVPNNETWORKMASK "24"
:global OVPNNETWORK "10.0.6.0/24"
:global OVPNIPADDRESS "10.0.6.254"
:global OVPNCLIENTIPADDRESS "10.0.6.253"

:global OVPNDHCPPOOLNAME "ovpn-dhcpPool"
:global OVPNDHCPPOOLRANGE "10.0.6.249-10.0.6.253"

:global USERNAME "ovpn-Client1"
:global PASSWORDUSERLOGIN "ovpn-Client1-Password"

## 2022.04.21 - it it really limited to 8characters?
## it is being used to secure the private key
:global PASSWORDCERTPASSPHRASE "12345678"
```

### Configuration - Certificate, Interface, Firewall
```shell
## generate a CA certificate
/certificate
add name=ca-template common-name="$CN" days-valid="$DAYSVALIDCA" \
  key-usage=crl-sign,key-cert-sign
sign ca-template ca-crl-host=127.0.0.1 name="$CN"
:delay 10

## generate a server certificate
/certificate
add name=server-template common-name="ovpn-server@$CN" days-valid="$DAYSVALIDSERVER" \
  key-usage=digital-signature,key-encipherment,tls-server
sign server-template ca="$CN" name="ovpn-server@$CN"
:delay 10

## create a client template
/certificate
add name="$OVPNCLIENTCERTIFICATETEMPLATENAME" common-name="$OVPNCLIENTCERTIFICATETEMPLATECOMMONNAME" \ days-valid="$DAYSVALIDCLIENT" key-usage=tls-client
:delay 5

## create IP pool
/ip pool
add name="$OVPNDHCPPOOLNAME" ranges="$OVPNDHCPPOOLRANGE"

## add VPN profile
/ppp profile
add local-address="$OVPNIPADDRESS" name="$OVPNPROFILENAME" \
  remote-address="$OVPNDHCPPOOLNAME" use-ipv6=no use-upnp=no use-compression=no use-mpls=no use-encryption=yes only-one=yes

## setup OpenVPN server
/interface ovpn-server server
set auth=sha512 certificate="ovpn-server@$CN" cipher=aes128,aes192,aes256 \
  default-profile="$OVPNPROFILENAME" mode=ip netmask="$OVPNNETWORKMASK" port="$PORT" \
  protocol=udp enabled=yes require-client-certificate=yes tls-version=only-1.2

## setup OpeVPN interface binding
/interface ovpn-server
add name="$USERNAME" user="$USERNAME"

## add a firewall rule
/ip firewall filter
add chain=input action=accept dst-port="$PORT" protocol=udp comment="OpenVPN - Allow"
add chain=input action=accept dst-port=53 protocol=udp src-address="$OVPNNETWORK" \
  comment="OpenVPN - Accept DNS requests from clients"
## the movement of the rules may not work in case there are default firewall rules
## it throws an failure: can not move builtin as the builtin 0 is passthrough for the fasttrack counters, then move the rule manually so it is right after the fasttrack rule
move [find comment="OpenVPN - Allow"] 0
move [find comment="OpenVPN - Accept DNS requests from clients"] 1

## Setup completed. Do not forget to create a user.
```

### Configuration - add user, export certificates
```shell
## add a user
/ppp secret
add local-address="$OVPNIPADDRESS" name="$USERNAME" \
  password="$PASSWORDUSERLOGIN" profile="$OVPNPROFILENAME" \ remote-address="$OVPNCLIENTIPADDRESS" service=ovpn

## generate a client certificate
/certificate
add name=client-template-to-issue copy-from="$OVPNCLIENTCERTIFICATETEMPLATENAME" \
  common-name="$USERNAME@$CN"
sign client-template-to-issue ca="$CN" name="$USERNAME@$CN"
:delay 10

/certificate
set "ovpn-server@$CN" trusted=yes

## export the CA, client certificate, and private key
/certificate
export-certificate "$CN" export-passphrase=""
export-certificate "$USERNAME@$CN" export-passphrase="$PASSWORDCERTPASSPHRASE"

## clear the console history to get rid of sensitive information
/console clear-history
```
At this stage the certificates should look like this
```shell
/certificate print
Flags: K - PRIVATE-KEY; L - CRL; A - AUTHORITY; I, R - REVOKED; T - TRUSTED
Columns: NAME, COMMON-NAME, FINGERPRINT
#       NAME                   COMMON-NAME            FINGERPRINT                                                     
0 KLA T MikroTik               MikroTik               thumbprint
1 K  IT ovpn-server@MikroTik        ovpn-server@MikroTik   thumbprint
2       ovpn-ClientTemplate        ovpnClientTemplate
3 K  I  ovpn-Client1@MikroTik  ovpn-Client1@MikroTik  thumbprint

```
NameOfyourMikrotikDevice - equals
```shell
/system identity get name
```
USERNAME equals the name of your first OpenVPN Client (in this example ovpn-Client1)<br><br>
The OpenVPN Server piece is done. Created certificates can be found in Files.<br>
cert_export_[nameOfyourMikrotikDevice] - CA cert<br>
cert_export_[$USERNAME].cert<br>
cert_export_[$USERNAME].key<br>

Download the exported certificates, and make use of them on the OpenVPN client device.

### Configuration - add static routes
Add static routes towards your client device, via your OpenVPN gateway Interface.
```shell
/ip route
add distance=1 dst-address=192.168.33.0/24 gateway="$USERNAME"
## add any extra routes towards the networks available behind the tunnel
```
On top of that **bring your firewall rules**.

### Check
There is great chance that at this stage the openVPN client is not configured. Once it is and the tunnel is set properly, then among the IP addresses, dynamically assigned IP should arise for the openVPN traffic.
```shell
/ip address print 
Flags: X - disabled, I - invalid, D - dynamic 
 #   ADDRESS            NETWORK         INTERFACE                         
 0   ;;; defconf
## IP address assigned to the bridge allows you managing the mikrotik via winbox
     192.168.88.1/24    192.168.88.0    bridge
## ether1 is the WAN interface, it is your uplink for the ovpn gateway
 1 D 172.16.253.253/24  172.16.253.0    ether1
## here is the IP address of the openVPN interface
 2 D 10.0.6.254/32      10.0.6.253      <ovpn-ovpn-Client1>
```

### Debug
```shell
/system logging add topics=ovpn,debug,!packet
/system rule print
/system logging remove numbers=[number of the ruke]
/system rule reset numbers=[number of the rule]
```

## Configuration - add another OpenVPN client
With the abovementioned configuration there is one server and one client, 1:1 approach. If the goal is to have more clients, 1:n approach then repeat the steps described below for the OpenVPN server configuration and customize each OpenVPN client individually once the configuration on the server side.

```shell
:global CN [/system identity get name]

:global OVPNCLIENTCERTIFICATETEMPLATENAME "ovpn-ClientTemplate"
## :global OVPNCLIENTCERTIFICATETEMPLATENAME "client-template"
##:global OVPNCLIENTCERTIFICATETEMPLATENAME "ovpn-ClientTemplate"
:global OVPNCLIENTCERTIFICATETEMPLATECOMMONNAME "ovpnClientTemplate"

:global OVPNPROFILENAME "ovpn-profile"
:global OVPNIPADDRESS "10.0.6.254"
:global OVPNCLIENTIPADDRESS "10.0.6.252"

## those parameters are being used in PPP Secrets configuration
:global USERNAME "ovpn-Client2"
:global PASSWORDUSERLOGIN "ovpn-Client2-Password"

## 2022.04.21 - should it be 8character long
## it is being used to secure the private key
## it is provided during the phase of setting up connection from the client
## unless it is removed with openssl, so the user does not have to put the password
## it applies to Desktop OS or FreshTomato, as up to my knowledge Mikrotik requires
## the password to be in place
:global PASSWORDCERTPASSPHRASE "87654321"
```

```shell
## add interface binding
/interface ovpn-server
add name="$USERNAME" user="$USERNAME"

## add a user
/ppp secret
add name=$USERNAME password="$PASSWORDUSERLOGIN" profile="$OVPNPROFILENAME" service=ovpn \
local-address="$OVPNIPADDRESS" remote-address="$OVPNCLIENTIPADDRESS"

## generate a client certificate
/certificate
add name=client-template-to-issue copy-from="$OVPNCLIENTCERTIFICATETEMPLATENAME" \
  common-name="$USERNAME@$CN"
sign client-template-to-issue ca="$CN" name="$USERNAME@$CN"
:delay 10

#export the certificate
/certificate
## it is not needed to export the CA cert again
## export-certificate "$CN" export-passphrase=""
export-certificate "$USERNAME@$CN" export-passphrase="$PASSWORDCERTPASSPHRASE"
```
Once certificates are exported 
```shell
/file print
# at this moment you should see
# those are the CA certificate and the certificate which should be bound with client's openVPN configuration
3 cert_export_MikroTik.crt               .crt  file     1188 
4 cert_export_ovpn-Client2@MikroTik.key  .key  file     1858 
5 cert_export_ovpn-Client2@MikroTik.crt  .crt  file     1168 
6 cert_export_ovpn-Client2@MikroTik.key  .key  file     1858 
7 cert_export_ovpn-Client2@MikroTik.crt  .crt  file     1168 
```

Download the exported certificates, for the another user, and make use of them on subsequent OpenVPN client device.

## Summary
I'm sure there are better ways doing it, but still it's a good starting point.<br>
It was tested on RB951G and CCR with ROS 7.3.1<br>
Last update: 2022.06.18