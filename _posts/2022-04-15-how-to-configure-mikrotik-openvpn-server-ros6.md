---
layout: post
title: "How to configure Mikrotik OpenVPN Server - ROS 6.X - TCP"
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
+ NTP server configured properly, so the time and date is in sync
+ OpenSSL (not a must)
+ A bit of Mikrotik knowledge

## Background

I'm not an expert of any means in that area, use it at your own risk.<br>
This example shows how to configure the OpenVPN server in semiautomated fashion, based on the Long Term Mikrotik router OS version 6.48.6 (as for the time of writing this blog post).
Newer releases of Router OS offers extra parameters which are not included within current 6.X version, configuration described within this blog post should work on ROS 7.X, never the less I'm not recommending using it that way.<br>
Current configuration will also work on 7.X releases, I'm sure that with the use of TLS 1.2 or more advanced hashing algirithms instead of sha1 it can be made more secure.<br>
OpenVPN tunnel can be established betwen the devices with different ROS major versions.

### NAT

Your mikrotik can be behind NAT provided the port on which your OpenVPN server is listening is forwarded on the router which your mikrotik device is connecting to, to get access to the Internet.

### OpenSSL

In this usecase, OpenSSL is being used to remove the password from the certificate key. This can be usefull when your client is FreshTomato based. For Mikrotik devices, password removal from the private key is **NOT** needed. Certificates are generated on the Mikrotik device itself, no need for EeasyRSA or OpenSSL being compiled or installed.

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

## Auth and Ciphers

Mikrotik equipped with RouterOS long term release 6.48.6, supports SHA1 or MD5 for Auth, as it goes for Ciphers it is blowfish 128 or AES128-256 as the cipher. If another types are needed, it should be updated to 7.X.

```shell
 #CCR 6.48.6
 /system logging add topics=ovpn,!debug
 /log print
 #ovpn,info : using encoding - AES-256-CBC/SHA1
 #when you finished disable ovpn logging
 /system logging add topics=ovpn,!debug disabled=yes
```

## Preparation - Dates

If the goal is that your self signed certificates expires in the same period of time, you may want a align with those which are alreay in place, and for that you may need to calculate amount of days remaining towards particular date. This is the way to count it, provided you have PowerShell in hand.

```powershell
# here we align with the expiration date of 19th of January 2025
$now = get-date
New-TimeSpan -Start $now -end $(get-date('01.19.2025'))
# at the time of writing this article it was 945 days left towards this date
(Get-Date).AddDays(945)
```

### Assumptions

+ you start from zero, and reset the configuration of the Mikrotik to it's defaults (It is not mandatory as you can make use of the commands and execute on top of your existing configuration, it should also work with some tweaks)
+ your uplink is the ether1, never the less it will also work if it is lte1 or wlan1 (If your uplink is not ether1, then you can add ether1 into the bridge - then all 5 ether ports can connect to your devices/switches, and modify the Interface list, by pointing the WAN from ether1 to lte1 or wlan1, at the end modify the dhcp client to listen on lte1 or wlan1)
+ the network ranges on the OpenVPN server side and the OpenVPN client *differs from each other* it may have an issue to work if both subnets are the same
+ once the openVPN is configured *add the correct routes on the device which plays the OpenVPN server, as well as on the OpenVPN client side* (routes should dst towards the networks on the other side of the tunnel and your gateway would be your openvpn interface)
+ the OpenVPN network range is within the 10.0.6.0/24 (here I assume that your router has 5 ethernet interfaces and the virtual interface which is openVPN is the next available network range, still there is another assumption that you are using the 24 network mask here)

### Prerequisites

+ DDNS name or static IP address on the WAN interface
+ Mikrotik RB/CCR device
+ the NTP server configured properly, so the time and date is in sync
+ details about the network segments on the other side of the tunnel, to configure the routes properly

### Configuration - defining variables

Open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window.

**script does not work if the passwords contains \ *backslash***

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
  remote-address="$OVPNDHCPPOOLNAME" use-encryption=yes use-mpls=no use-compression=no use-upnp=no only-one=yes

## setup OpenVPN server
/interface ovpn-server server
set auth=sha1 certificate="ovpn-server@$CN" cipher=aes128,aes192,aes256 \
  default-profile="$OVPNPROFILENAME" mode=ip netmask="$OVPNNETWORKMASK" port="$PORT" \
  enabled=yes require-client-certificate=yes

## setup OpeVPN interface binding
/interface ovpn-server
add name="$USERNAME" user="$USERNAME"

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

Add interface binding, generate and export the certificates

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
```

Download the exported certificates, for the another user, and make use of them on the OpenVPN client device.

## Summary

I'm sure there are better ways doing it, but still it's a good starting point.

It was tested on RB951G and CCR with ROS 6.48.6

Last update: 2022.06.18
