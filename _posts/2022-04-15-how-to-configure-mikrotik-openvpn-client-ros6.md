---
full-width: true
layout: post
title: "How to configure Mikrotik OpenVPN client - ROS 6.X - TCP"
permalink: "/how-to-configure-mikrotik-openvpn-client-ros6/"
subtitle: "Router OS 6.48.6"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [Networking, Mikrotik ,OpenVPN]
categories: [Networking, Mikrotik, OpenVPN]
---
In this scenario the regular traffic is routed through the Internet, where the target networks defined within the static routes, are traversed over the OpenVPN tunnel. The connection takes place between two Mikrotik devices, where the client reach the Internet over lte1 interface.

## Prerequisites

+ DNS alias or IP address of your OpenVPN server
+ Mikrotik RB/CCR device
+ NTP server configured properly, so the time and date is in sync
+ The Mikrotik OpenVPN server configured [How to configure Mikrotik OpenVPN Server - RouterOS 6](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server-ros6/)

## Backgroupnd

+ openVPN tunnel between the server with ROS version 6 and client with ROS 7 will work
+ the network ranges on both ends of your openVPN tunnel should differ (it's important as if you reset your devices and apply default configuration, initially on both ends there is 192.168.88.X network, it should be changed at least on one device, to make it work)

+ [github gist](https://gist.github.com/ea1het/3a168a88a6a8c86ee26e71a83a2d71d9)
+ [wawrus](https://www.wawrus.pl/technologia/konfiguracja-openvpn-na-mikrotiku)
+ [grzegorzkowalik](https://grzegorzkowalik.com/mikrotik-openvpn-server/)

+ if the device is reset to it's defailt configuration apply the below settings

## Howto

Plug the LTE stick to your Mikrotik device equipped with USB port, then connect it to the power adapter.

The procedure contains few steps which should be executed in following order

1. import the files exported during the OpenVPN server configuration (File -> Upload)
2. import the certificates
3. create PPP Profile
4. create PPP Interface
5. adress lists will be created dynamically once the OpenVPN is established, no need to create it manually
6. add routes (dst is the target network you reach over the tunnel, gateway is the ovpn interface)

### Configuration

Convert signal from LTE and pass to ether ports

```shell
## disable wireless interface
/interface wireless
set [ find default-name=wlan1 ] band=2ghz-b/g/n channel-width=20/40mhz-XX country=poland distance=indoors frequency=auto installation=indoor mode=ap-bridge \
    ssid=951G wireless-protocol=802.11
## modify the ip pool range
/ip pool
add name=default-dhcp ranges=192.168.33.10-192.168.33.254
## add the ether1 to the bridge, this give you 5 operational ports within the abovemention network range
/interface bridge port
add bridge=bridge comment=defconf ingress-filtering=no interface=ether1
add bridge=bridge comment=defconf ingress-filtering=no interface=ether2
add bridge=bridge comment=defconf ingress-filtering=no interface=ether3
add bridge=bridge comment=defconf ingress-filtering=no interface=ether4
add bridge=bridge comment=defconf ingress-filtering=no interface=ether5
add bridge=bridge comment=defconf ingress-filtering=no interface=wlan1
## set the lte1 as WAN interface
/interface list member
add comment=defconf interface=lte1 list=WAN
## set IP Address on the bridge
/ip address
add address=192.168.33.1/24 comment=defconf interface=bridge network=192.168.33.0
## disable dhcp-client on the ether1 interface, the dhcp-client on lte1 will be created dynamically for you
/ip dhcp-client
add comment=defconf disabled=yes interface=ether1
## define the range for the dhcp-server along with your DNS servers
/ip dhcp-server network
add address=192.168.33.0/24 comment=defconf dns-server=1.1.1.2,1.1.1.1 gateway=\
    192.168.33.1 netmask=24
## set the dns server globally on the device as well
/ip dns
set allow-remote-requests=yes servers=1.1.1.2,1.1.1.1
```

In case you cut yourself off from the device, just refresh your endpoint IP address or connect to the mikrotik device via it's MAC address, as this option is not disabled with it's default configuration.

### Configuration - defining variables

Open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window.**The overall logic does not work if the passwords contains \ *backslash***

```shell
:global CN [/system identity get name]
:global OVPNSERVERPORT 4911
:global OVPNSERVERFQDN "XXX.YYY.ZZZ"

## the USERNAME goes hand in hand with the OpenVPN PPP Secret
:global USERNAME "ovpn-Client1"
:global PASSWORDUSERLOGIN "ovpn-Client1-Password"

:global OVPNCLIENTINTERFACENAME "ovpn-centrala"
:global OVPNPROFILENAME "ovpn-profile"
##:global OVPNIPADDRESS "10.0.6.254"
:global OVPNCLIENTIPADDRESS "10.0.6.253"
## 2022.04.21 - does the limit it 8character long?
## this is the passphrace for the private key copied from the openVPN server configuration
:global PASSWORDCERTPASSPHRASE "12345678"
```

Now it's time to upload the certificates which was prepared for the client during the openVPN Server setup.

+ Winbox -> Files -> Upload three certificates (cert_export_Mikrotik.crt, cert_export_ovpn-Client1@Mikrotik.crt, cert_export_ovpn-Client1@Mikrotik.key)

```shell
/file print 
Columns: NAME, TYPE, SIZE, CREATION-TIME
#  NAME                                                TYPE       SIZE      CREATION-TIME       
3  cert_export_MikroTik.crt                            .crt file  1188      apr/15/2022 22:30:58
4  cert_export_ovpn-Client1@MikroTik.crt               .crt file  1168      arp/15/2022 22:30:58
5  cert_export_ovpn-Client1@MikroTik.key               .key file  1858      apr/15/2022 22:30:58
```

Once the files are uploaded it's time to import the certificates

```shell
## passphrase is empty, when asked just hit enter
/certificate import name="ovpn-server-CA" file-name=cert_export_MikroTik.crt
passphrase: 
     certificates-imported: 1
     private-keys-imported: 0
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

## passphrase is empty, when asked just hit enter
/certificate import name="ovpn-Client1" file-name=cert_export_ovpn-Client1@MikroTik.crt
     certificates-imported: 1
     private-keys-imported: 0
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

/certificate print 
Flags: L - CRL; A - AUTHORITY; T - TRUSTED
Columns: NAME, COMMON-NAME
#     NAME           COMMON-NAME          
0 LAT ovpn-server-CA MikroTik             
1   T ovpn-Client1   ovpn-Client1@MikroTik

## import private key
## passphrase equals the one set during the openVPN server configuration
/certificate import name="ovpn-Client1-key" file-name=cert_export_ovpn-Client1@MikroTik.key passphrase="$PASSWORDCERTPASSPHRASE"
     certificates-imported: 0
     private-keys-imported: 1
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

/certificate print 
Flags: K - PRIVATE-KEY; L - CRL; A - AUTHORITY; T - TRUSTED
Columns: NAME, COMMON-NAME
#      NAME           COMMON-NAME          
0  LAT ovpn-server-CA MikroTik             
1 K  T ovpn-Client1   ovpn-Client1@MikroTik
```

When certificates are imported, continue with further configuration .

### Configuration - PPP Profile and Interface

```shell
## configure PPP Profile
/ppp profile add name="$OVPNPROFILENAME" change-tcp-mss=yes only-one=yes use-compression=no use-encryption=yes use-mpls=no use-upnp=no

## configure PPP Interface 
/interface ovpn-client add name="$OVPNCLIENTINTERFACENAME" connect-to="$OVPNSERVERFQDN" port="$OVPNSERVERPORT" profile="$OVPNPROFILENAME" certificate="$USERNAME" user="$USERNAME" password="$PASSWORDUSERLOGIN" add-default-route=no auth=sha1 cipher=aes256 disabled=no verify-server-certificate=yes
```

The traffic should be passed provided the firewall rules allows it.
In case the routes for some reason are not configured dynamically, add static routes

### Configuration - add static routes

+ On the OpenVPN Client device - On top of existing configuration add static routes towards the networks which are nated behind your OpenVPN server.
+ On the OpenVPN Server device - On top of existing configuration add static routes towards the networks which are nated behind your OpenVPN client.

```shell
## dst-address is the network on the other side of the tunnel
/ip route
add disabled=no dst-address=192.168.88.0/24 gateway="$OVPNCLIENTINTERFACENAME" routing-table=main suppress-hw-offload=no
## add any extra routes towards networks which should be made reachable through the VPN tunnel
```

On top of that **bring your firewall rules**.

## Debug

In case something does not work, or you get the TLS error, [check this first(https://openvpn.net/faq/tls-error-tls-key-negotiation-failed-to-occur-within-60-seconds-check-your-network-connectivity/).

```shell
/system logging add topics=ovpn,debug,!packet
/system rule print
/system logging remove numbers=[number of the rule]
/system rule reset numbers=[number of the rule]
```

## Summary

I'm sure there are better ways doing it, but still it's a good starting point.

It was tested on RB951G, ROS 6.48.6

Last update: 2022.06.18
