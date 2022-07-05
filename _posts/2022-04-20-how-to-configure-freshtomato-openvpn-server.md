---
layout: post
title: "How to configure FreshTomato OpenVPN server - TLS 1.3"
permalink: "/how-to-configure-freshtomato-openvpn-server/"
subtitle: "Certificate complemented with UserName and Password"
cover-img: /assets/img/cover/img-cover-tunnel.jpg
thumbnail-img: /assets/img/thumb/img-thumb-tomato.jpg
share-img: /assets/img/cover/img-cover-tunnel.jpg
tags: [HomeLab ,Networking ,OpenVPN ,FreshTomato]
categories: [HomeLab ,Networking ,OpenVPN ,FreshTomato]
---
This post describe how to setup OpenVPN server on Freshtomato. There are so many versions, that new commers may feel overwhelmed, and there are plenty traps during configuration. I'm not an expert by any means in those areas, using this in my homelab and actually this is the sole purpose of the overall usecase, that someone can make use of second hand 25$ cpu ARMv7 based router, and setup an OpenVPN on top of it to get access to the homelab, when roaming, or expose it's LAN resources towards the oher FreshTomato based OpenVPN client or mobile devices etc.<br>

## Prerequisites
+ FreshTomato VPN compatible device, version [2022.3](https://freshtomato.org/downloads/freshtomato-arm/2022/2022.3/K26ARM/)
+ Worth to mention that devices behind the tunnel should be in different subnets, otherwise there may be routing issues with the regular configuration. Here the 172.16.88.0 subnet is being used for the LAN bridge on the device playing OpenVPN server role
+ Time should be in sync for the OpenVPN server and clients (make use of NTP)

## Background
At lot is going on in security area. It seems that this blog post will be obsolete soon, never the less at least for home lab usecase it may bring some value, apart from being educational.
0. Some says that after succesfull configuration of the OpenVPN Server/Client reboot the device, in order to get rid of TLS authentication issue. Based on my experience, this was not necessary as the tunnel was established immediatelly after the puzzles has been composed into the overall picture. From the other hand, I've been experiencing TLS authentication issues, when the PassPhrases has been set for CA and Server certificate.
1. Upgrade all your FreshTomato servers and clients with the same software release, and clear the NVRAM, before performing the setup. I can not guarantee that the configuration mentioned here will work with other versions of the OpenVPN than the one mentioned here. It's so dynamic and there is plenty of depreciation of features or parameters betwen versions.
2. Some says that backward compatibility of the certs left much to be desired. That's why in your lab usecase it's much easier to have them regenerated than deep dive with troubleshooting. It turned out that EasyRSA 3.1.0 serves the OpenVPN 2.5.6 very well as it goes for the certificates. If it does not then you may experience TLS handshake errors, or incorrect certificates configurations.
3. Routers which are short with NVRAM, are not a limitation, you can make use of JFFS or other mounted storage on your router to store them. Typically devices with 32KB of NVRAM may encounter this issue, as certificates can consume 14KB of space. This means you can still make use of RT-N16U if you have one somewhere in your wardrobe.
4. It looks that SHA1 has been cracked since early 2019, this is one of the reasons here within FreshTomato SHA256/SHA512 has been used. Mikrotik with RouterOS 7 also gives that option. It seems that FreshTomato is using SHA1 to hash the passwords for the openVPN usernames.
5. TCP tunnels requires MTU settings, UDP tunnels can make use of mssfix option. Small amount of fragmentations should increase the overall thorughput of the tunnel. Within this configuration mssfix is not being used.
6. Start with some reading first. It will make your life much easier.
+ OpenVPN [wiki](https://openvpn.net/community-resources/how-to/#pki) - definitelly worth reading
+ OpenVPN [Custom Configuration](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/) parameters
+ OpenVPN [forum](https://forums.openvpn.net/viewforum.php?f=31)
+ DD-WRT [wiki](https://wiki.dd-wrt.com/wiki/index.php/OpenVPN)
+ DD-WRT [OpenVPN Server setup guide](https://forum.dd-wrt.com/phpBB2/viewtopic.php?t=318795) - you need an account on the forum to see the guide. This entry was one of the major source of knowledge for the post.
+ DD-WRT [youtube](https://www.youtube.com/watch?v=dwrR18_xO_Q) - which is following along the abovementioned guide, it's really helpfull as well
+ DD-WRT [VPN Easy Way V24+](https://wiki.dd-wrt.com/wiki/index.php/VPN_(the_easy_way)_v24+)
+ FreshTomato [wiki](https://wiki.freshtomato.org/doku.php/openvpn_server)
+ Sekurak [OpenVPN practical implementation](https://sekurak.pl/praktyczna-implementacja-sieci-vpn-na-przykladzie-openvpn/)
+ HowtoForge [Install OpenVPN server and client with easyRSA 3 on Centos 8](https://www.howtoforge.com/tutorial/how-to-install-openvpn-server-and-client-with-easy-rsa-3-on-centos-8/)
+ EasyRSA on [archlinux](https://wiki.archlinux.org/title/Easy-RSA) wiki
+ EasyRSA on [gentoo](https://wiki.gentoo.org/wiki/Create_a_Public_Key_Infrastructure_Using_the_easy-rsa_Scripts#Create_CA_certificate) wiki
+ Github [Intro To PKI](https://github.com/OpenVPN/easy-rsa/blob/master/doc/Intro-To-PKI.md)

## ToDo
This blog post is far from being perfect, I'm not an expert in any means in those areas.
+ aside from cetrificate authentication add the [username and password](https://wiki.dd-wrt.com/wiki/index.php/OpenVPN)
+ CRL - how to generate and make it work
+ kill switch in case of VPN dropout like [here](https://protonvpn.com/support/vpn-freshtomato-router/) - this is especially usefull when the vpn connection is your default gateway for the client.
+ set the firewall rules, that traffic is allowed or denied based on the ovpn USERNAME

## Howto
First and foremost enable the SSH access, so you can get into your tomato via SSH.
Open webbrowser and navigate to you https://FreshTomatoMgmtIPAddress/admin-access.asp

```shell
Enable at Startup    - unchecked
Extended MOTD        - checked
Remote Access        - unchecked
Remote Forwarding    - unchecked
Port                 - [SSH port number]             
Allow Password Login - checked
#if your preference is to use key authentication then configure it accordingly
Authorized Keys      - empty
```

Start the SSH Deamon, and login via SSH to the device. Starting this point of time you can establish a WinScp connection with making use of the SCP protocol to download/upload certificates.

```powershell
$freshTomatoMgmtIPAddress = "172.16.88.1"
$freshTomatoSSHPortNumber = 2288
Start-Process "ssh" -ArgumentList "root@$freshTomatoMgmtIPAddress -p $freshTomatoSSHPortNumber"
# put the password from the regular user which is being used to login via web browser
# https://172.16.88.1/admin-access.asp -> slide to the bottom -> username / password
```

## FreshTomato - Configuration - Basic Settings
It turns out, that the VPN Server is distributing the IP addresses starting the bottom of the subnet range, so the *tun21* network interface of the OpenVPN server has an address of 10.0.6.241/28, where the *tun11* interface has the 10.0.6.242/28.
VPN Tunneling -> OpenVPN Server -> Basic

```shell
Start with WAN                  - checked
Interface Type                  - TUN
Protocol                        - UDP
# by default the Port number is 1149
Port                            - [OVPNSERVERPORTNUMBER]
Firewall                        - Automatic
Authorization Mode              - TLS
# TLS control channel details can be found here: https://openvpn.net/vpn-server-resources/tls-control-channel-security-in-openvpn-access-server/
TLS control channel security    - Encrypt Channel V2
(tls-auth/tls-crypt)	

Auth digest	                    - SHA512
# netmask is defining the amount of devices, here you have 14 which with this protocol will exagerate the CPU anyway
VPN subnet/netmask              - 10.0.6.240 | 255.255.255.240
```
## FreshTomato - Configuration - Advanced Settings
Chacha is less CPU intensive
+ [Do the Chacha](https://blog.cloudflare.com/do-the-chacha-better-mobile-performance-with-cryptography/) - better mobile performance with cryptography
+ [It takes two to Chacha](https://blog.cloudflare.com/it-takes-two-to-chacha-poly/)
VPN Tunneling -> OpenVPN Server -> Advanced

```shell
Poll Interval - 0  (in minutes, 0 to disable)
Push LAN0 (br0) to clients      - checked
Push LAN1 (br1) to clients      - grayed out
Push LAN2 (br2) to clients      - grayed out
Push LAN3 (br3) to clients      - grayed out
Direct clients to               - unchecked
redirect Internet traffic
Respond to DNS                  - unchecked

# the default Data Ciphers are: CHACHA20-POLY1305:AES-128-GCM:AES-256-GCM:AES-128-CBC:AES-256-CBC
# OpenVPN 2.5.6 the first compatible cipher is being used, CBC is left for the backward compatibility
# with Mikrotik 6.X, if you have very current OpenVPN clients and modern hardware leave CHACHA and GCM
Data ciphers                    - CHACHA20-POLY1305:AES-128-GCM:AES-256-GCM:AES-256-CBC
# use RiverBed or SD-WAN instead, never the less if you can afford for such it, I'm sure
# you won't bother with Compression on top of OpenVPN protocol
Compression                     - Disabled
TLS Renegotiation Time          - -1  (in seconds, -1 for default)
Manage Client-Specific Options	- unchecked
Allow User/Pass Auth            - checked
Allow Only User/Pass            - unchecked
(without cert Auth)
```

As the Allow User/Pass Auth is checked, put the login and password for the client, it should goes hand in hand with USERNAME and PASSWORDUSERLOGIN from the [blog post](https://makeitcloudy.pl/how-to-configure-freshtomato-openvpn-client/) and the *FreshTomato OpenVPN Client - Basic Tab* configuration.<br>
UserName and Password complement certificates. Once you check the Allow User/Pass Auth, you are shown with the GUI which allows you specifying those valuse.

## FreshTomato - Configuration - Advanced Settings - Custom Configuration
Paste following content into the Custom Configuration. Some says that since OpenVPN 2.4 it is not needed to use Diffie Hellman anymore, and the configuration can base on the eliptic curve cryptography, which indeed works with OpenVPN 2.5.6.
+ In case you need the log to be more verbose increase the natural number next to the verb parameter
+ Mute parameter specify the the Log, logs n consecutive messages in the same category
+ Remember about adding routes towards the networks on the other side of the VPN tunnel

```shell
# TODO: check how to generate the CRL
# CRL is not configured yet
# using the CA management of your choice, generate a Certificate Revocation List
# store the file somewhere on the OpenVPN server filesystem and point to it with the configuration
# crl /full/path/to/crl.pem

dh none
ecdh-curve secp384r1
# verb is a natural number between 0 and 11, the higher the number the move verbose it is
# 5 is quite verbose. starting 6 it comes into debug
verb 4
mute 10

# destination network netmask ovpn-gateway
#route 10.0.5.0 255.255.255.0 vpn_gateway
route 192.168.28.0 255.255.255.0 vpn_gateway

# OPTIONALLY (do not needed with current configuration, informational purposes only):
# This parameter can be used for the backward compatibility with Mikrotik 6.X
# never the less this is not enough, as then Auth would have to be changed to SHA1
# I have not tested this, but this is at least one of the aspects which needs to be taken into account
# data-ciphers-fallback AES-256-CBC
```

## FreshTomato - Configuration - SSL certificates and keys
Apart from the UserName and Password, the certificates are also to be used. FreshTomato 2022.3 uses OpenVPN 2.5.6 along with OpenSSL 1.1.1o.<br>
TLS Authorization Mode which was picked, requires certificates. In order to met that requirement following elements are needed (on VPN server those are stored in NVRAM)
```shell
FileName    Keep it private Purpose                                     Usedby
ca.crt      No              Root CA Certificate (Certificate Authority) Server,Client (it signs the certificates)
# OpenVPN server can verify signature without needing access to the CA private key itself
# that's why it can be stored on the key signing machine only, and stored safely
# as it is the most sensitive key in the entire PKI, if you loose control on it your PKI is compromised
ca.key      Yes             Root CA key (it signs all certificates)     key signing machine only
# FreshTomato does NOT use TLS with elliptic curve cryptography, that's why you must configure Diffie-Helman
dh.pem      No              Diffie Hellman                              Server
# certificate and private key for the OpenVPN server
server.crt  No              OpenVPN Server Certificate                  Server
server.key  Yes             OpenVPN Server Certificate key              Server
# certificate and private key for the first client
client.crt  No              OpenVPN Client Certificate                  Client
client.key  Yes             OpenVPN Client Certficate key               Client
```

The ca.key is only used to sign the certificates. Keep it secure and do **NOT** copy to the server nor clients.<br>

The FreshTomato [wiki](https://openvpn.net/community-resources/how-to/) states that the OpenVPN server certificates can be created directly in FreshTomato GUI using the **Generate Key** button or with making use of EasyRSA, but do **NOT** use it due to very weak entropy, try using bare metal to generate certificates.In this scenario EasyRSA 3 is in use (I was not very fortunate with the GUI button), neither with previous releases of the EeasyRSA 2.X.<br>
+ [EasyRSA 3](https://github.com/OpenVPN/easy-rsa)

### FreshTomato - Configuration - EasyRSA 3 - Configure CA Certificates
Open PowerShell session with Administrative permissions. Some says that creating the certificates with EasyRSA on Windows has it's [drawbacks](https://wiki.dd-wrt.com/wiki/index.php/VPN_(the_easy_way)_v24+) - check the paragraph *Creating Certificates Using Easy RSA in Windows*. I have no observe this, once certs were created, was able to use those immediatelly.<br>
OpenVPN version available on FreshTomato 2022.3 is 2.5.6, which goes hand in hand with EasyRSA 3.X. So we should be covered from that perspective.

```powershell
Set-Location -Path 'C:\Temp\EasyRSA-3.1.0'
.\openssl.exe version
OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

# Run EasyRSA to start generating certificates
.\EasyRSA-Start.bat
# Welcome to the EasyRSA 3 Shell for Windows.
# Easy-RSA 3 is available under a GNU GPLv2 license.

# Invoke './easyrsa' to call the program. Without commands, help is displayed.
```

Generate folder structure for certificates. Remember that OpenVPN 2.4 bring the requirement for the minimum key size of 2048, so older keys with lower size will not work.

```powershell
# EasyRSA Shell
./easyrsa init-pki
#* Notice:
#  init-pki complete; you may now create a CA or requests.
#  Your newly created PKI dir is:
#  * C:/Program Files/EasyRSA-3.1.0/pki
#* Notice:
#  IMPORTANT: Easy-RSA 'vars' file has now been moved to your PKI above.
```

If your goal is to update the certificates which may secure the web services traffic in the same point of time, and you need to compute the amount of days until specific date, it can be done this way

```powershell
# here we align with the expiration date of 19th of January 2025
$now = get-date
New-TimeSpan -Start $now -end $(get-date('01.19.2025'))
# at the time of writing this article it was 1004 days left towards this date
(Get-Date).AddDays(1004)
```

Uset Notepad++, to edit the *C:\Temp\EasyRSA-3.1.0\pki\vars* file, uncomment following entries

```powershell
Get-ChildItem -Path 'C:\Program Files\EasyRSA-3.1.0\pki'
# Edit vars file, uncomment following entries
set_var EASYRSA_REQ_COUNTRY	"PL"
set_var EASYRSA_REQ_PROVINCE	"Marshland"
set_var EASYRSA_REQ_CITY	"Pristine"
set_var EASYRSA_REQ_ORG	"MakeITcloudy"
#set_var EASYRSA_REQ_EMAIL	"me@example.net"
set_var EASYRSA_REQ_OU	"SurprisedItWorks"
set_var EASYRSA_KEY_SIZE	2048
# EASYRSA_CA_EXPIRE defines the amount of days till the ca.crt certificate expires, by default it's 10years
#set_var EASYRSA_CA_EXPIRE	3650
# EASYRSA_CERT_EXPIRE defines the amount of days till the server.crt certificate expires, by default it is 825days
set_var EASYRSA_CERT_EXPIRE	1004
```

Still in the EasyRSA Shell, execute next commands to build the CA. The var file has been customized already, those parameters will be used during this phase. The common name of the is worth to be wrote down, never the less this is not this value which is being used as the Common Name check during the phased of certificate verification.

```shell
./easyrsa build-ca nopass
#Using Easy-RSA configuration from: C:/Temp/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

#Using configuration from C:/Temp/EasyRSA-3.1.0/pki/safessl-easyrsa.cnf.init-tmp
#....+...+..+++..+++
#....+....+..++..+++
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
#Common Name (eg: your user, host, or server name) [Easy-RSA CA]:FreshTomato-CA

#* Notice:

#CA creation complete and you may now import and sign cert requests.
#Your new CA certificate file for publishing is at:
#C:/Temp/EasyRSA-3.1.0/pki/ca.crt
```

At this stage you have
+ ca.crt - this file is being used on the OpenVPN Server as well as OpenVPN client
+ ca.key - this file remains private

### FreshTomato - Configuration - EasyRSA 3 - Request Server Certificate
It's time to generate OpenVPN Server certificate.
+ the *nopass* parameter is being used, so you are **not** asked for the PEM pass phrase. If this parameter is ommited, then there is a need to put the same passphrase which was used during the CA certificate creation, never the less here for the CA creation, the *nopass* switch was also used.
* write down, the Common Name value, as it will be used on the OpenVPN Client configuration for the verification process of the certificate Name

```shell
#EasyRSA Shell
./easyrsa gen-req FreshTomato-ovpnServer nopass
#* Notice:
#Using Easy-RSA configuration from: C:/Temp/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)
#.......+...+++*.+...
#.......+...+..+...+.
#..........++++++++++
#-----
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
#Common Name (eg: your user, host, or server name) [FreshTomato-ovpnServer]:FreshTomato-ovpnServer
#* Notice:

#Keypair and certificate request completed. Your files are:
#req: C:/Temp/EasyRSA-3.1.0/pki/reqs/FreshTomato-ovpnServer.req
#key: C:/Temp/EasyRSA-3.1.0/pki/private/FreshTomato-ovpnServer.key
```

### FreshTomato - Configuration - EasyRSA 3 - Sign the CSR of Server Certificate
Create the OpenVPN Server certificate, this certificate is being used only on the OpenVPN Server configuration and is not being shared with the Clients.

```shell
#EasyRSA Shell
./easyrsa sign-req server FreshTomato-ovpnServer
#* Notice:
#Using Easy-RSA configuration from: C:/Temp/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

#You are about to sign the following certificate.
#Please check over the details shown below for accuracy. Note that this request
#has not been cryptographically verified. Please be sure it came from a trusted
#source or that you have verified the request checksum with the sender.

#Request subject, to be signed as a server certificate for 1004 days:

#subject=
#    commonName                = FreshTomato-ovpnServer

#Type the word 'yes' to continue, or any other input to abort.
#  Confirm request details: yes

#Using configuration from C:/Temp/EasyRSA-3.1.0/pki/safessl-easyrsa.cnf.init-tmp
#90250000:error:0700006C:configuration file routines:NCONF_get_string:no value:crypto/conf/conf_lib.c:315:group=<NULL> #name=unique_subject
#Check that the request matches the signature
#Signature ok
#The Subject's Distinguished Name is as follows
#commonName            :ASN.1 12:'FreshTomato-ovpnServer'
#Certificate is to be certified until Jan 18 15:52:15 2025 GMT (1004 days)

#Write out database with 1 new entries
#Data Base Updated

#* Notice:
#Certificate created at: C:/Temp/EasyRSA-3.1.0/pki/issued/FreshTomato-ovpnServer.crt
#EasyRSA Shell
```

### FreshTomato - Configuration - EasyRSA 3 - Request Client certificate
Repeat those steps for each OpenVPN client separatelly. It is also possible to make it later, provided you had safely backed up your ca.crt and ca.key, and have **not** run *./easyrsa init-pki* during the period of time when the certificates are valid, as this command is wiping out existing keys.
+ the *nopass* parameter causes that it is not needed to remove the password from the private key, in case this is not being used some extra tweak would be necessary

```shell
#EasyRSA
# with current configuraiton it is not needed to execute this, as during the certificate creation
# the parameter nopass is being used
openssl rsa -passin pass:[SERVER_PEM_PASSPHRASE] -in FreshTomato.key -out FreshTomatoNoPass.key
```

+ the Common Name, from current section is the one which is exposed within the OpenVPN Server Configuration status field on the *Current List* and *Routing Table* among the connected clients

```shell
./easyrsa gen-req ovpn-Client1 nopass
#* Notice:
#Using Easy-RSA configuration from: C:/Temp/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)
#.+..........+..
#..+.......+..+.
#-----
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
#Common Name (eg: your user, host, or server name) [ovpn-Client1]:ovpn-Client1
#* Notice:

#Keypair and certificate request completed. Your files are:
#req: C:/Temp/EasyRSA-3.1.0/pki/reqs/ovpn-Client1.req
#key: C:/Temp/EasyRSA-3.1.0/pki/private/ovpn-Client1.key
```

### FreshTomato - Configuration - EasyRSA 3 - Sign the CSR of the Client Certificate
Create the OpenVPN Client certificate, it is being used on the OpenVPN Client configuration and identifies particular client.
+ Common Name - is the identificator of the connected client
```shell
#EasyRSA Shell
./easyrsa sign-req client ovpn-Client1
#* Notice:
#Using Easy-RSA configuration from: C:/Temp/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

#You are about to sign the following certificate.
#Please check over the details shown below for accuracy. Note that this request
#has not been cryptographically verified. Please be sure it came from a trusted
#source or that you have verified the request checksum with the sender.

#Request subject, to be signed as a client certificate for 1004 days:

#subject=
#    commonName                = ovpn-Client1

#Type the word 'yes' to continue, or any other input to abort.
#  Confirm request details: yes

#Using configuration from C:/Temp/EasyRSA-3.1.0/pki/safessl-easyrsa.cnf.init-tmp
#Check that the request matches the signature
#Signature ok
#The Subject's Distinguished Name is as follows
#commonName            :ASN.1 12:'ovpn-Client1'
#Certificate is to be certified until Jan 18 15:57:09 2025 GMT (1004 days)

#Write out database with 1 new entries
#Data Base Updated

#* Notice:
#Certificate created at: C:/Temp/EasyRSA-3.1.0/pki/issued/ovpn-Client1.crt
```

## Depreciated - FreshTomato - Configuration - EasyRSA - Generate DH
*No need to run this, this section can be skipped - it's left here only for informational purposes, in case someone got used to it, and during the years was using DH.<br>
Diffie Helman is being used for the safe key exchange. EasyRSA key size was defined above as 2048 bits, if it is changed anytime during the validity of the cetificates generated, you should recreate the certificates from scratch.<br>
+ [Some says](https://wiki.archlinux.org/title/OpenVPN) that starting OpenVPN 2.4.8 to specify the eliptic curve in server configuration. Otherwise the server would fail to recognize the curve type, resulting an authentication errors.
+ OpenVPN 2.4 brought Eliptical Curve Diffie Hellman to generate the key (provided the client also supports this). More details about it when googling for EDCSA certificates. That's why within the Advanced Tab and Custom configuration the following settings were set.<br>

```shell
dh none
ecdh-curve secp384r1
```

Depending from your hardware, it is the most lengthly process as it goes for the time of generating DH.

```shell
./easyrsa gen-dh

#* Notice:
#Using Easy-RSA configuration from: C:/Program Files/EasyRSA-3.1.0/pki/vars
#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

#Generating DH parameters, 2048 bit long safe prime
#..............................................
#.........++*++
#* Notice:
#DH parameters of size 2048 created at C:/Program Files/EasyRSA-3.1.0/pki/dh.pem
```

At this stage there are following files:

```shell
C:/Program Files/EasyRSA-3.1.0/pki/ca.crt
C:/Program Files/EasyRSA-3.1.0/pki/private/ca.key
# req is not needed for the configuration
# C:/Program Files/EasyRSA-3.1.0/pki/reqs/FreshTomato-ovpnServer.req
# C:/Program Files/EasyRSA-3.1.0/pki/reqs/ovpn-Client1.req
# within the OpenVPN Server configuration the NoPass.key is being used
C:/Program Files/EasyRSA-3.1.0/pki/private/FreshTomato-ovpnServer.key
C:/Program Files/EasyRSA-3.1.0/pki/issued/FreshTomato-ovpnServer.crt
C:/Program Files/EasyRSA-3.1.0/pki/issued/ovpn-Client1.crt
# in our example we make use Elliptic Curve, so no need for DH
#C:/Program Files/EasyRSA-3.1.0/pki/dh.pem
```

## FreshTomato - Configuration - OpenVPN Server
VPN Tunneling -> OpenVPN Server ->

```shell
# FreshTomato GUI within the Keys tab, states the following
#  Optional, only used for client certificate generation.
#  Uncrypted (-nodes) private keys are supported.
#  TODO: check if the OpenVPN tunnel is established without the ca.key pasted here..

# only paste the certificate content between and including -BEGIN-- --END-- sections to preserve space on NVRAM
# do not paste the descriptive details from the certificate content which is available above those sections
Certificate Authority Key - paste the ca.key, including -----BEGIN ENCRYPTED PRIVATE KEY----- and -----END ENCRYPTED PRIVATE KEY-----
Certificate Authority     - paste the ca.crt, including -----BEGIN CERTIFICATE----- and -----END CERTIFICATE-----
Server Certificate        - paste FreshTomato-ovpnServer.crt, including -----BEGIN CERTIFICATE----- and -----END CERTIFICATE-----
Server Key                - paste FreshTomato-ovpnServer.key, including -----BEGIN PRIVATE KEY----- and -----END PRIVATE KEY-----
# In case you generated the dh.pem anyway, there is no need
# to paste dh.pem content into OpenVPN Server Keys, as in current config Elliptic Curves are being used, so the
# Diffie Helman Parameters are left blank within the FreshTomato GUI
#Diffie Helman Parameters  - paste dh.pem, uncluding -----BEGIN DH PARAMETERS----- and -----END DH PARAMETERS-----
```


1. Save configuration. At this stage the NVRAM consumption should increase, in my case 60,49% of NVRAM is utilized.
2. Press the *Start Now* button and within the *Status* section, you should be shown with new elements like, Current List, Routing Table, General Statistics.
3. In order to proof that the tunnel has been set up, go to your SSH session which you established at the very beggining and execute

```shell
ifconfig
## among interfaces like br0, eth0, eth1, lo, vlan1, vlan2 you will also see tun21
tun21      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
           inet addr:10.0.6.241  P-t-P:10.0.6.241  Mask:255.255.255.240
           UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
           RX packets:430 errors:0 dropped:0 overruns:0 frame:0
           TX packets:6287 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000
           RX bytes:26550 (25.9 KiB)  TX bytes:502741 (490.9 KiB)
## this will confirm that at least another virtual interface arose, and  it's IP comes from the VPN Subnet in VPN Tunneling Basic settings configuration pane.
```

Establishing the tunnel between FreshTomato devices, with *verb 4* in the custom config, the log indicates it's 3 seconds.

## Final thoughts
+ ca.crt is common for the clients, until it gets renewed on the OpenVPN server device
+ ovpn-ClientX.crt and ovpn-ClientX.key is generaged individually for each OpenVPN client and separatelly shared with anyone you'd like to share your VPN with
+ Following files are shared with each client, apart from that each client has it's own UserName and Password.
```shell
C:/Program Files/EasyRSA-3.1.0/pki/ca.crt #it's the same for each client
#those two goes in pair and are uniques for each client
C:/Program Files/EasyRSA-3.1.0/pki/issued/ovpn-Client1.crt
C:/Program Files/EasyRSA-3.1.0/pki/private/ovpn-Client1.key
```

On top of that, drop your firewall rules.

## Summary
That's it.<br>
Tested on FreshTomato 2022.3, it uses OpenVPN 2.5.6, OpenSSL 1.1.1o, on RT-N18U.<br>
Certificates generated with: EasyRSA 3.1.0, OpenSSL 3.0.3 on Windows<br>
Last update: 2022.07.01