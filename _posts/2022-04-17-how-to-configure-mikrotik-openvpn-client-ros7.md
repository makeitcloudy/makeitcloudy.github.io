---
layout: post
title: "How to configure mikrotik as OpenVPN client"
permalink: "/how-to-configure-mikrotik-openvpn-client-ros7/"
subtitle: "Router OS 7.3.1"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,Mikrotik ,OpenVPN]
categories: [HomeLab ,Networking ,Mikrotik, OpenVPN]
---
*2022.04.17 - Draft*
In this scenario the regular traffic is routed through the Internet, where the target networks defines within the routes, are traversed over the OpenVPN tunnel. The connection takes place between two Mikrotik devices, where the client reach the Internet over lte1 interface. 

## Prerequisites
+ DNS alias or IP address of your OpenVPN server
+ Mikrotik RB/CCR device
+ OpenVPN Server configured
+ The Mikrotik OpenVPN server configured [How to configure Mikrotik OpenVPN Server](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server/)

## Howto
Plug the LTE stick to your Mikrotik device equipped with USB port, then connect it to the power adapter.<br>
The procedure contains few steps which should be executed in following order
1. import the files exported during the OpenVPN server configuration (File -> Upload)
2. import the certificates
3. adress lists will be created dynamically once the OpenVPN is established, no need to create it manually
4. create PPP Profile
5. (not needed) create PPP Secret
5. create PPP Interface
6. add routes (dst is the target network you reach over the tunnel)

+ openVPN tunnel between the server with ROS version 6 and client with ROS 7 will work
### Configuration - client preparation
If you start from zero, with Mikrotik configuration reset back to it's defaults (on both sides VPN Server and Client), then the network range for those destinations should differ. Initially, when the Mikrotik configuration is reset it assigns the 192.168.88.0/24 network on top of the bridge. It needs to be changed
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

Now it's time to upload the certificates which was prepared for the client during the openVPN Server setup.
+ Winbox -> Files -> Upload three certificates (cert_export_Mikrotik.crt, cert_export_ovpn-Client1@Mikrotik.crt, cert_export_ovpn-Client1@Mikrotik.key)
```shell
[user@MikroTik] > file print 
Columns: NAME, TYPE, SIZE, CREATION-TIME
#  NAME                                                TYPE       SIZE      CREATION-TIME       
3  cert_export_MikroTik.crt                            .crt file  1188      jun/18/2022 22:30:58
4  cert_export_ovpn-Client1@MikroTik.crt               .crt file  1168      jun/18/2022 22:30:58
5  cert_export_ovpn-Client1@MikroTik.key               .key file  1858      jun/18/2022 22:30:58
```
Once the files are uploaded it's time to import the certificates
```shell
## passphrase is empty, when asked just hit enter
[user@MikroTik] > certificate import name="ovpn-server-CA" file-name=cert_export_MikroTik.crt
passphrase: 
     certificates-imported: 1
     private-keys-imported: 0
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

## passphrase is empty, when asked just hit enter
[user@MikroTik] > certificate import name="ovpn-Client1" file-name=cert_export_ovpn-Client1@MikroTik.crt
     certificates-imported: 1
     private-keys-imported: 0
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

[user@MikroTik] > certificate print 
Flags: L - CRL; A - AUTHORITY; T - TRUSTED
Columns: NAME, COMMON-NAME
#     NAME          COMMON-NAME          
0 LAT CA            MikroTik             
1   T ovpn-Client1  ovpn-Client1@MikroTik

## import private key
## passphrase equals the one set during the openVPN server configuration
[user@MikroTik] > certificate import name="ovpn-Client1-key" file-name=cert_export_ovpn-Client1@MikroTik.key passphrase="$PASSWORDCERTPASSPHRASE"
     certificates-imported: 0
     private-keys-imported: 1
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

[piotrek@MikroTik] > certificate print 
Flags: K - PRIVATE-KEY; L - CRL; A - AUTHORITY; T - TRUSTED
Columns: NAME, COMMON-NAME
#      NAME          COMMON-NAME          
0  LAT CA            MikroTik             
1 K  T ovpn-Client1  ovpn-Client1@MikroTik
```
When certificates are imported, continue with further configuration depending from your ROS version.

## Router OS 7.3.1
This section is dedicated for the Router OS version 7.

## Router OS 6.48.6
This section is dedicated for the Router OS version 6.
+ [github gist](https://gist.github.com/ea1het/3a168a88a6a8c86ee26e71a83a2d71d9)
+ [wawrus](https://www.wawrus.pl/technologia/konfiguracja-openvpn-na-mikrotiku)
+ [grzegorzkowalik](https://grzegorzkowalik.com/mikrotik-openvpn-server/)

### Configuration - ROS 6.X - defining variables
At this stage, certificates created during the configuration of the Mikrotik OpenVPN Server, are already imported. Once this is done, open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window.<br>
**script does not work if the passwords contains \ *backslash***

```shell
:global CN [/system identity get name]
## the USERNAME may go hand in hand with the USERNAME set during the configuration of the VPNServer
:global USERNAME "Client1"
:global OVPNPROFILENAME "ovpn-profile"
:global OVPNIPADDRESS "10.0.6.254"
## 2022.04.21 - does the limit it 8character long?
:global PASSWORDCERTPASSPHRASE "12345678"
:global PASSWORDUSERLOGIN "clientPassword"
:global OVPNCLIENTIPADDRESS "10.0.6.253"
```
### Configuration - ROS 6.X - Execute this piece of code on device which acts as OpenVPN client
```shell
## configure PPP Profile
ppp profile add name="$OVPNPROFILENAME" change-tcp-mss=yes only-one=yes use-compression=no use-encryption=yes use-ipv6=no use-mpls=no use-upnp=no

# this can be skipped as secret is not needed with the certificate based authentication
## configure PPP Secret (add a user)
## local address = address of your VPN server
## remote address = client address connecting VPN service
## /ppp secret add name=$USERNAME password="$PASSWORDUSERLOGIN" profile="$OVPNPROFILENAME"
## service=ovpn local-address="$OVPNIPADDRESS" remote-address="$OVPNCLIENTIPADDRESS"

## configure PPP Interface 
interface ovpn-client add connect-to=xxx.xxx.xxx.xxx add-default-route=no auth=sha1 certificate=client disabled=no user=vpnuser password=vpnpass name=myvpn profile=OVPN-client

## ....

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