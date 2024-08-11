---
full-width: true
layout: post
title: "How to configure Mikrotik OpenVPN client - ROS 7.X - UDP"
permalink: "/how-to-configure-mikrotik-openvpn-client-ros7/"
subtitle: "Router OS 7.3.1"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-openvpn.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [Networking, Mikrotik, OpenVPN]
categories: [Networking, Mikrotik, OpenVPN]
---
This blog post is a follow-up of another blog post which was describing [how to configure Mikrotik OpenVPN client on ROS 6.X](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-client-ros6/)

## To Do

+ improve the configuration that it allows verifying the server certificate, at this stage with current example, it is not establishing the tunnel if it is the case, so here it is not required, which seems to be a security risk. If one knows how to fix this, please feel free to comment this blog post.

## Prerequisites

+ DNS alias or IP address of your OpenVPN server
+ Mikrotik RB/CCR device
+ NTP server configured properly, so the time and date is in sync
+ OpenVPN Server configured, certificates exported
+ The Mikrotik OpenVPN server configured [How to configure Mikrotik OpenVPN Server](https://makeitcloudy.pl/how-to-configure-mikrotik-openvpn-server/)

### Configuration - defining variables

At this stage, certificates created during the configuration of the Mikrotik OpenVPN Server, are already imported. Once this is done, open Mikrotik terminal, change variables below if needed, and paste into Mikrotik terminal window.

**script does not work if the passwords contains \ *backslash***

```shell
:global CN [/system identity get name]
## OVPNSERVERPORT variable is the port number on which the OpenVPN Server is listening
## it goes hand in hand with the PORT variable being used during the OpenVPN Server config
:global OVPNSERVERPORT 4911
## OVPNSERVERFQDN is the Fully Qualified domain name of the OpenVPN Server interface
## most propably this is just the WAN link of the device playing that role
:global OVPNSERVERFQDN "XXX.YYY.ZZZ"

## the USERNAME goes hand in hand with the OpenVPN PPP Secret
:global USERNAME "ovpn-Client1"
:global PASSWORDUSERLOGIN "ovpn-Client1-Password"

:global OVPNCLIENTINTERFACENAME "ovpn-centrala"
:global OVPNPROFILENAME "ovpn-profile"
##:global OVPNIPADDRESS "10.0.6.254"
:global OVPNCLIENTIPADDRESS "10.0.6.253"
## 2022.04.21 - does the limit it 8character long?
## PASSWORDCERTPASSPHRASE is the passphrase for the private key copied from the openVPN server
## configuration - it is used during the phase of establishing the connection between the client
## and the server
:global PASSWORDCERTPASSPHRASE "12345678"
```

### Configuration - importing certificates

Now it's time to upload the certificates which was prepared for the client during the openVPN Server setup.

+ Winbox -> Files -> Upload three certificates (cert_export_Mikrotik.crt,cert_export_ovpn-Client1@Mikrotik.crt, cert_export_ovpn-Client1@Mikrotik.key)

```shell
/file print 
Columns: NAME, TYPE, SIZE, CREATION-TIME
#  NAME                                                TYPE       SIZE      CREATION-TIME       
3  cert_export_MikroTik.crt                            .crt file  1188      jun/18/2022 22:30:58
4  cert_export_ovpn-Client1@MikroTik.crt               .crt file  1168      jun/18/2022 22:30:58
5  cert_export_ovpn-Client1@MikroTik.key               .key file  1858      jun/18/2022 22:30:58
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
#     NAME          COMMON-NAME          
0 LAT CA            MikroTik             
1   T ovpn-Client1  ovpn-Client1@MikroTik

## import private key
## passphrase equals the one set during the openVPN server configuration
/certificate import name="ovpn-Client1-key" file-name=cert_export_ovpn-Client1@MikroTik.key passphrase="$PASSWORDCERTPASSPHRASE"
     certificates-imported: 0
     private-keys-imported: 1
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0

/certificate/print 
Flags: K - PRIVATE-KEY; L - CRL; A - AUTHORITY; T - TRUSTED
Columns: NAME, COMMON-NAME
#      NAME           COMMON-NAME          
0  LAT ovpn-server-CA MikroTik             
1 K  T ovpn-Client1   ovpn-Client1@MikroTik
```

When certificates are imported, continue with further configuration depending from your ROS version.

### Configuration - final tweaking

```shell
## configure PPP Profile
## chage-tcp-mss not used here, seems to me that it is an option for tcp protocol
## but not sure about it
/ppp profile add name="$OVPNPROFILENAME" only-one=yes use-encryption=yes use-ipv6=no use-compression=no use-mpls=no use-upnp=no

# this can be skipped as secret is not needed with the certificate based authentication
## configure PPP Secret (add a user)
## local address = address of your VPN server
## remote address = client address connecting VPN service
## /ppp secret add name=$USERNAME password="$PASSWORDUSERLOGIN" profile="$OVPNPROFILENAME"
## service=ovpn local-address="$OVPNIPADDRESS" remote-address="$OVPNCLIENTIPADDRESS"

## configure PPP Interface
## with current configuration if the vertify server certificate is in use for some reason
## it can not establish the VPN connection, blind shoot is that it's due to CRL
## the lower the ciphers the more throughput should be made available
## it may be beneficial if you pass through the tunnel thick protocols
/interface ovpn-client add name="$OVPNCLIENTINTERFACENAME" connect-to="$OVPNSERVERFQDN" port="$OVPNSERVERPORT" profile="$OVPNPROFILENAME" protocol=udp mode=ip certificate="$USERNAME" user="$USERNAME" password="$PASSWORDUSERLOGIN" add-default-route=no auth=sha512 cipher=aes256 disabled=no tls-version=only-1.2 use-peer-dns=no
```

Routes should be added dynamically once the tunnel is established. In case for some reason are not, static routes can be added.

### Configuration - Routes

+ On the OpenVPN Client device - On top of existing configuration add static routes towards the networks which are nated behind your OpenVPN server.
+ On the OpenVPN Server device - On top of existing configuration add static routes towards the networks which are nated behind your OpenVPN client. 

```shell
/ip route
add disabled=no dst-address=192.168.88.0/24 gateway="$OVPNCLIENTINTERFACENAME" routing-table=main suppress-hw-offload=no
## add any extra routes towards networks which should be made reachable through the VPN tunnel
```

On top of that **bring your firewall rules**.

## Debug

In case something does not work, or you get the TLS error, [check this first](https://openvpn.net/faq/tls-error-tls-key-negotiation-failed-to-occur-within-60-seconds-check-your-network-connectivity/).

```shell
/system logging add topics=ovpn,debug,!packet
/system rule print
/system logging remove numbers=[number of the rule]
/system rule reset numbers=[number of the rule]
```

## Summary

I'm sure there are better ways doing it, but still it's a good starting point.

It was tested on RB951G, ROS 7.3.1

Last update: 2022.04.17
