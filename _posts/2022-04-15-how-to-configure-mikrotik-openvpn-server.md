---
layout: post
title: "How to configure Mikrotik OpenVPN Server"
permalink: "/how-to-configure-mikrotik-openvpn-server/"
subtitle: "When you are properly equipped, it's quite straighforward"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
categories: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
---
*As of 2022.04.15 - draft*
Setting up another virtual interface on Mikrotik is not that difficult provided you know how to do it. OpenSSL installed on your Desktop can be handy to remove the password protection from key in case you are making use of it on the FreshTomato client or you don't want your OpenVPN client to provide it during the phase of establishing a connection to your VPN server.

## Prerequisites
+ Mikrotik Routerboard/CCR device
+ OpenSSL (not a must)
+ a bit of Mikrotik knowledge

## Background
ROS 7.3.1 offers following options which does not seem to be available with 6.48.8.
+ TLS 1.2 and Auth sha256 and sha512 for PPP OpenVPN interface
+ IPv6 for PPP profile protocols
+ ROS 7.x offers wireguard which may be a good alternative for openVPN.

As of now (2022.06) the ROS 6.X configuration work on ROS 7.X, never the less those extra parameters which can be passed into newer ROS configuration, won't be parsed properly on older ROS.

## Preparation - OpenSSL
In current context the OpenSSL is being used to remove the password from the certificate key. Then such cert can be used later on (not in Mikrotik usecase, but DesktopOS configuration or FreshTomato) - if you are not using those scenarios, then the OpenSSL section can be skipped.
### Linux
In case the OpenSSL is not available on your machine, but you can quickly spin up a VM, I've wrote a blog post [how to compile OpenSSL from sources](https://makeitcloudy.pl/how-to-compile-openssl-from-sources/).
### Windows
I'm sure there is a way to compile the OpenSSL on Windows (probably [this](https://github.com/openssl/openssl/blob/master/NOTES-WINDOWS.md) is the way to go), never the less there is a quicker way to make use of OpenSSL on this platform as well, especially if you do a bit of coding and make use of GIT already. OpenSSL exist in the following directory *C:\Program Files\Git\usr\bin* you may use it from there, or add it to *$env:PATH* and run it directly from the commandline. At the time of writing this I'm on

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
If there is some reason that you'd like to calculate the amount of days towards specific date, when your certificates should expire, and you don't just use X years from now here is the way to count it.

```powershell
$now = get-date
New-TimeSpan -Start $now -end $(get-date('01.19.2025'))
(Get-Date).AddDays(945)
```
## Howto 
This example shows, how to configure Mikrotik OpenVPN server it in semiautomated fashion.
### Assumption 
+ you start from zero, and reset the configuration of the mikrotik to it's defaults (It is not mandatory as you can make use of the commands and execute on top of your existing configuration, it should also work with some tweaks)
+ your uplink is the ether1, never the less it will also work if it is lte1 or wlan1 (If your uplink is not ether1, then you can add it into the bridge - then all 5 ether ports can connect to your devices/switches, and modify the Interface list, by pointing the WAN from ether1 to lte1 or wlan1, at the end modify the dhcp client to listen on lte1 or wlan1)
+ the network ranges on the OpenVPN server side network topology and the OpenVPN client differs from each other
+ once the openVPN is configured add the correct routes on the device which plays the OpenVPN server, as well as on the OpenVPN client side (routes should dst towards the networks on the other side of the tunnel)
+ the OpenVPN network range is within the 10.0.6.0/24

## Router OS 7.3.1
This section is dedicated for the Router OS version 7.

### Configuration - ROS 7.X - defining variables
.

### Configuration - ROS 7.X - Execute this piece of code on the device which act as OpenVPN Server
.

### Configuration - ROS 7.X - add user and export certificates
.

Download the exported certificates, and make use of them on the OpenVPN client device.

## Router OS 6.48.6
This section is dedicated for the Router OS version 6.
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

:global OVPNPROFILENAME "ovpn-profile"
:global OVPNIPADDRESS "10.0.6.254"
:global OVPNNETWORK "10.0.6.0/24"
:global OVPNNETWORKMASK "24"

:global OVPNDHCPPOOLNAME "dhcpPool-ovpn"
:global OVPNDHCPPOOLRANGE "10.0.6.249-10.0.6.253"

:global USERNAME "ovpn-Client1"
:global OVPNCLIENTIPADDRESS "10.0.6.253"
## 2022.04.21 - it it really limited to 8characters?
:global PASSWORDCERTPASSPHRASE "12345678"
:global PASSWORDUSERLOGIN "clientPassword"
```
### Configuration - ROS 6.X - Execute this piece of code on the device which act as OpenVPN Server
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
add name=client-template common-name="client" days-valid="$DAYSVALIDCLIENT" \
  key-usage=tls-client
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
add chain=input action=accept dst-port="$PORT" protocol=tcp comment="Allow OpenVPN"
add chain=input action=accept dst-port=53 protocol=udp src-address="$OVPNNETWORK" \
  comment="Accept DNS requests from VPN clients"
## the movement of the rules may not work in case there are default firewall rules
## it throws an failure: can not move builtin as the builtin 0 is passthrough for the fasttrack counters, then move the rule manually so it is right after the fasttrack rule
move [find comment="Allow OpenVPN"] 0
move [find comment="Accept DNS requests from VPN clients"] 1

## Setup completed. Do not forget to create a user.
```

### Configuration - ROS 6.X - add user and export certificates
```shell
## add a user
/ppp secret
add local-address="$OVPNIPADDRESS" name="$USERNAME" password="$PASSWORDUSERLOGIN" profile="$OVPNPROFILENAME" remote-address="$OVPNCLIENTIPADDRESS" service=ovpn

## generate a client certificate
/certificate
add name=client-template-to-issue copy-from=client-template \
  common-name="$USERNAME@$CN"
sign client-template-to-issue ca="$CN" name="$USERNAME@$CN"
:delay 10

## at this stage the server certifcate should be trusted
##  ca - KLAT
##  server - KI
##  client - KI
## where in it is
##  ca - KLAT
##  server - KIT
##  client - KI
/certificate
set "server@$CN" trusted=yes

## export the CA, client certificate, and private key
/certificate
export-certificate "$CN" export-passphrase=""
export-certificate "$USERNAME@$CN" export-passphrase="$PASSWORDCERTPASSPHRASE"
```
The OpenVPN Server piece is done. Created certificates can be found in Files.<br>
cert_export_[nameOfyourMikrotikOvpnServer] - CA cert<br>
cert_export_[nameOfyourClient].cert<br>
cert_export_[nameOfyourClient].key<br>

+ nameOfyourMikrotikOvpnServer - equals
```shell
/system identity get name
```
+ nameOfyourClient equals the USERNAME parameter

Download the exported certificates, and make use of them on the OpenVPN client device.

## Summary
I'm sure there are better ways doing it, but still it's a good starting point.<br>
It was tested on RB951G and CCR with ROS 6.48.6<br>
Not tested yet with ROS 7.3.1<br>
Last update: 2022.06.18