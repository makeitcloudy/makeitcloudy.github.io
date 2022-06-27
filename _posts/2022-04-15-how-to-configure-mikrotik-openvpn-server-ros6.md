---
layout: post
title: "How to configure Mikrotik OpenVPN Server - ROS 6.X"
permalink: "/how-to-configure-mikrotik-openvpn-server-ros6/"
subtitle: "Router OS 6.48.6"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
categories: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
---
Setting up another virtual interface on Mikrotik is not that difficult provided you know how to do it. OpenSSL installed on your Desktop can be handy to remove the password protection from key in case you are making use of it on the FreshTomato client or you don't want your OpenVPN client to provide it during the phase of establishing a connection to your VPN server.

## Prerequisites
+ DDNS configuration on top of your dynamic IP address or static IP address
+ Mikrotik Routerboard/CCR device
+ OpenSSL (not a must)
+ A bit of Mikrotik knowledge

## Background
This example shows how to configure the OpenVPN server in semiautomated fashion, based on the Long Term Mikrotik router OS version 6.48.6 (as for the time of writing this blog post).
Newer releases of Router OS offers extra parameters which are not included within current 6.X version.<br>
Current configuration will also work on 7.X releases, never the less I'm sure that with the use of TLS 1.2 or more advanced hashing algirithms instead of sha1 it can be made more secure.<br>
### NAT
Your mikrotik can be behind NAT provided the port on which your OpenVPN server is listening is forwarded on the router which your mikrotik device is connecting to, to get access to the Internet.
### OpenSSL
In this usecase, OpenSSL is being used to remove the password from the certificate key. This can be usefull when your client is FreshTomato based. For Mikrotik devices, it is **NOT** used. 
### OpenSSL on Linux
This arcicle [how to compile OpenSSL from sources](https://makeitcloudy.pl/how-to-compile-openssl-from-sources/) provides the details how to compile it on a linux machine.
### OpenSSL on Windows
I'm sure there is a way to compile the OpenSSL on Windows (seems [this](https://github.com/openssl/openssl/blob/master/NOTES-WINDOWS.md) is the way to go), never the less there is a quicker way to make use of OpenSSL on this platform as well, especially if you do a bit of coding and make use of GIT already. OpenSSL exist in the following directory *C:\Program Files\Git\usr\bin* you may use it from there, or add it to *$env:PATH* and run it directly from the commandline. At the time of writing this I'm on

```console
git --version
git version 2.36.1.windows.1
```
and it contains the OpenSSL 1.1.1o
```console
openssl.exe
OpenSSL> version
OpenSSL 1.1.1o  3 May 2022
```
Alternatively compiled version for Windows can be downloaded from [slproweb.com](https://slproweb.com/products/Win32OpenSSL.html)

## Preparation - Dates
If the goal is that your self signed certificates expires in the same period of time, you may want a align with those which are alreay in place, and for that you may need to calculate amount of days remaining towards particular date. This is the way to count it, provided you have PowerShell in hand.

```powershell
# here we align with the expiration date of 19th of January 2025
$now = get-date
New-TimeSpan -Start $now -end $(get-date('01.19.2025'))
# at the time of writing this article it was 945 days left towards this date
(Get-Date).AddDays(945)
```
## Howto 
I'm not an expert of any means in that area, use it at your own risk.<br>
### Assumption 
+ you start from zero, and reset the configuration of the Mikrotik to it's defaults (It is not mandatory as you can make use of the commands and execute on top of your existing configuration, it should also work with some tweaks)
+ your uplink is the ether1, never the less it will also work if it is lte1 or wlan1 (If your uplink is not ether1, then you can add ether1 into the bridge - then all 5 ether ports can connect to your devices/switches, and modify the Interface list, by pointing the WAN from ether1 to lte1 or wlan1, at the end modify the dhcp client to listen on lte1 or wlan1)
+ the network ranges on the OpenVPN server side and the OpenVPN client *differs from each other* it may have an issue to work if both subnets are the same
+ once the openVPN is configured *add the correct routes on the device which plays the OpenVPN server, as well as on the OpenVPN client side* (routes should dst towards the networks on the other side of the tunnel and your gateway would be your openvpn interface)
+ the OpenVPN network range is within the 10.0.6.0/24 (here I assume that your router has 5 ethernet interfaces and the virtual interface which is openVPN is the next available network range, still there is another assumption that you are using the 24 network mask here)

## Router OS 6.48.6
This section is dedicated for the Router OS version 6, it will also work on ROS 7 (I'm not recommending using it on ROS 7, it can be made more secure with extra params available on ROS 7)
### Configuration - ROS 6.X - defining variables
Open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window.<br>
**script does not work if the passwords contains \ *backslash***

```shell
:global CN [/system identity get name]
:global PORT 4911

:global DAYSVALIDCA 945
## it does not make sense for me that the SERVER is bigger/longer than CA which is used to sign the server cert
:global DAYSVALIDSERVER 945
:global DAYSVALIDCLIENT 365

:global OVPNCLIENTCERTIFICATETEMPLATENAME "ovpnClientTemplate"
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
add name=server-template common-name="server@$CN" days-valid="$DAYSVALIDSERVER" \
  key-usage=digital-signature,key-encipherment,tls-server
sign server-template ca="$CN" name="server@$CN"
:delay 10

## create a client template
/certificate
add name="$OVPNCLIENTCERTIFICATETEMPLATENAME" common-name="OVPNCLIENTCERTIFICATETEMPLATECOMMONNAME" \ days-valid="$DAYSVALIDCLIENT" key-usage=tls-client
:delay 5

## create IP pool
/ip pool
add name="$OVPNDHCPPOOLNAME" ranges="$OVPNDHCPPOOLRANGE"

## add VPN profile
/ppp profile
add local-address="$OVPNIPADDRESS" name="$OVPNPROFILENAME" \
  remote-address="$OVPNDHCPPOOLNAME" use-encryption=yes use-mpls=no use-compression=no use-upnp=no only-one=yes

## setup OpenVPN server
/interface ovpn-server server
set auth=sha1 certificate="server@$CN" cipher=aes128,aes192,aes256 \
  default-profile="$OVPNPROFILENAME" mode=ip netmask="$OVPNNETWORKMASK" port="$PORT" \
  enabled=yes require-client-certificate=yes

## add a firewall rule
/ip firewall filter
add chain=input action=accept dst-port="$PORT" protocol=tcp comment="OpenVPN - Allow"
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
add name=client-template-to-issue copy-from=client-template \
  common-name="$USERNAME@$CN"
sign client-template-to-issue ca="$CN" name="$USERNAME@$CN"
:delay 10

/certificate
set "server@$CN" trusted=yes

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
1 K  IT server@MikroTik        server@MikroTik        thumbprint
2       client-template        ovpnClientTemplate
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

### Configuration - Routes
On top of that add the routes towards your client device, via your OpenVPN gateway Interface.
### Debug
```shell
/system logging add topics=ovpn,debug,!packet
/system rule print
/system logging remove numbers=[number of the ruke]
/system rule reset numbers=[number of the rule]
```

## Summary
I'm sure there are better ways doing it, but still it's a good starting point.<br>
It was tested on RB951G and CCR with ROS 6.48.6<br>
Last update: 2022.06.18