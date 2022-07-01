---
layout: post
title: "How to configure FreshTomato OpenVPN server"
permalink: "/how-to-configure-freshtomato-openvpn-server/"
subtitle: "EasyRSA 3.1.0, OpenSSL 3.0.3, FreshTomato 2022.3"
cover-img: /assets/img/cover/img-cover-tunnel.jpg
thumbnail-img: /assets/img/thumb/img-thumb-tomato.jpg
share-img: /assets/img/cover/img-cover-tunnel.jpg
tags: [HomeLab ,Networking ,OpenVPN ,FreshTomato]
categories: [HomeLab ,Networking ,OpenVPN ,FreshTomato]
---
This post describe how to setup OpenVPN server on Freshtomato.

## Prerequisites
+ FreshTomato compatible device, with VPN version of the firmware installed on top of it, 2022.3 version can be downloaded [here](https://freshtomato.org/downloads/freshtomato-arm/2022/2022.3/K26ARM/)
+ Modified subnet from the detault one, in this example 172.16.88.1 is used (not needed but worth to remember that the devices behind OpenVPN server, and OpenVPN client needs to be in different subnets)

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
VPN Tunneling -> OpenVPN Server -> Basic

```shell
Start with WAN                  - checked
Interface Type                  - TUN
Protocol                        - UDP
Port                            - [OVPNSERVERPORTNUMBER]
Firewall                        - Automatic
Authorization Mode              - TLS
 
TLS control channel security - Disabled
(tls-auth/tls-crypt)	

Auth digest	                    - SHA512
VPN subnet/netmask              - 10.0.6.240 | 255.255.255.240
```
## FreshTomato - Configuration - Advanced Settings
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
# CHACHA20-POLY1305:AES-128-GCM:AES-256-GCM:AES-128-CBC:AES-256-CBC
Data ciphers                    - CHACHA20-POLY1305:AES-128-GCM:AES-256-GCM:AES-256-CBC
Compression                     - Disabled
TLS Renegotiation Time          - -1  (in seconds, -1 for default)
Manage Client-Specific Options	- unchecked
Allow User/Pass Auth            - unchecked
Custom Configuration	
```

## FreshTomato - Configuration - SSL certificates and keys
In order to setup the VPN you need following elements
```shell
FileName    Keep it private Purpose                                     Usedby
ca.crt      No              Root CA Certificate (Certificate Authority) All
ca.key      Yes             Root CA key (it signs all certificates)     key signing machine only
dh.pem      No              Diffie Hellman                              Server
server.crt  No              OpenVPN Server Certificate                  Server
server.key  Yes             OpenVPN Server Certificate key              Server
client.crt  No              OpenVPN Client Certificate                  Client
client.key  Yes             OpenVPN Client Certficate key               Client
```

All of them can be created with the use of [EasyRSA](https://github.com/OpenVPN/easy-rsa).

## FreshTomato - Configuration - EasyRSA - Configure CA Certificates

Open PowerShell session with Administrative permissions.
```powershell
Set-Location -Path 'C:\Program Files\EasyRSA-3.1.0'
PS C:\Program Files\EasyRSA-3.1.0> .\openssl.exe version
OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

# generate CA certificate
PS C:\Program Files\EasyRSA-3.1.0> .\EasyRSA-Start.bat
# Welcome to the EasyRSA 3 Shell for Windows.
# Easy-RSA 3 is available under a GNU GPLv2 license.

# Invoke './easyrsa' to call the program. Without commands, help is displayed.

# EasyRSA Shell
./easyrsa init-pki
#* Notice:
#  init-pki complete; you may now create a CA or requests.
#  Your newly created PKI dir is:
#  * C:/Program Files/EasyRSA-3.1.0/pki
#* Notice:
#  IMPORTANT: Easy-RSA 'vars' file has now been moved to your PKI above.
```

Indeed it was moved, to confirm that in separate powershell windows can run

```powershell
Get-ChildItem -Path 'C:\Program Files\EasyRSA-3.1.0\pki'
# Edit vars file, uncomment following entries
set_var EASYRSA_REQ_COUNTRY	"PL"
set_var EASYRSA_REQ_PROVINCE	"Marshland"
set_var EASYRSA_REQ_CITY	"Fertile"
set_var EASYRSA_REQ_ORG	"MakeITcloudy"
#set_var EASYRSA_REQ_EMAIL	"me@example.net"
set_var EASYRSA_REQ_OU	"niche"
set_var EASYRSA_KEY_SIZE	2048
# EASYRSA_CA_EXPIRE defines the amount of days till the ca.crt certificate expires, by default it's 10years
#set_var EASYRSA_CA_EXPIRE	3650
# EASYRSA_CERT_EXPIRE defines the amount of days till the server.crt certificate expires, by default it is 825days
set_var EASYRSA_CERT_EXPIRE	825
```

Still in the EasyRSA Shell, execute next commands which will bring you closer to the point of time where certificates will be created

```shell
./easyrsa build-ca
#* Notice:
#Using Easy-RSA configuration from: C:/Program Files/EasyRSA-3.1.0/pki/vars
#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)
# Enter New CA Key Passphrase: [CA_KEY_PASSPHRASE]
# Re-Enter New CA Key Passphrase: [CA_KEY_PASSPHRASE]

#Using configuration from C:/Program Files/EasyRSA-3.1.0/pki/safessl-easyrsa.cnf.init-tmp
#...+..+++++
# Enter PEM pass phrase: [CA_PEM_PASSPHRASE]
# Verifying - Enter PEM pass phrase: [CA_PEM_PASSPHRASE]
#-----
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
# Common Name (eg: your user, host, or server name) [Easy-RSA CA]: [TYPE_SOME_STRING_HERE]

#* Notice:

#CA creation complete and you may now import and sign cert requests.
#Your new CA certificate file for publishing is at:
#C:/Program Files/EasyRSA-3.1.0/pki/ca.crt
```

At this stage you have
+ ca.crt - this file is being used on the OpenVPN Server as well as OpenVPN client
+ ca.key - this file remains private

## FreshTomato - Configuration - EasyRSA - Configure Server Certificate
It's time to generate OpenVPN Server certificate.
+ once you are asked for the PEM pass phrase, put the same passphrase which was used during the CA certificate creation

```shell
#EasyRSA Shell
# ./easyrsa build-server-full FreshTomato
#* Notice:
#Using Easy-RSA configuration from: C:/Program Files/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

#..+...+.+..+..........+...
#..+.......++++++++++++++++
# Enter PEM pass phrase: [SERVER_PEM_PASSPHRASE]
# Verifying - Enter PEM pass phrase: [SERVER_PEM_PASSPHRASE]
#-----
#* Notice:

#Keypair and certificate request completed. Your files are:
#req: C:/Program Files/EasyRSA-3.1.0/pki/reqs/FreshTomato.req
#key: C:/Program Files/EasyRSA-3.1.0/pki/private/FreshTomato.key

#Using configuration from C:/Program Files/EasyRSA-3.1.0/pki/safessl-easyrsa.cnf.init-tmp
#Enter pass phrase for C:/Program Files/EasyRSA-3.1.0/pki/private/ca.key: [CA_PEM_PASSHPHRASE]
#74300000:error:0700006C:configuration file routines:NCONF_get_string:no value:crypto/conf/conf_lib.c:315:group=<NULL> #name=unique_subject
#Check that the request matches the signature
#Signature ok
#The Subject's Distinguished Name is as follows
#commonName            :ASN.1 12:'FreshTomato'
#Certificate is to be certified until Oct  3 20:59:30 2024 GMT (825 days)

#Write out database with 1 new entries
#Data Base Updated

#* Notice:
#Certificate created at: C:/Program Files/EasyRSA-3.1.0/pki/issued/FreshTomato.crt
```

## FreshTomato - Configuration - EasyRSA - Remove password from Server Certificate

```shell
#EasyRSA Shell
cd pki/private
#EasyRSA Shell
ls
#FreshTomato.key  ca.key

#EasyRSA Shell
openssl rsa -passin pass:[SERVER_PEM_PASSPHRASE] -in FreshTomato.key -out FreshTomatoNoPass.key
#writing RSA key

#EasyRSA Shell
ls
#FreshTomato.key        FreshTomatoNoPass.key  ca.key
```

## FreshTomato - Configuration - EasyRSA - Generate DH
Diffie Helman is being used for the safe key exchange
```shell
# Need to go out from pki/private to the original location where the easyrsa is located
#EasyRSA Shell
cd ../..
#EasyRSA Shell
ls
#COPYING.html            README.html             libssl-3-x64.dll
#COPYING.md              README.quickstart.html  openssl-easyrsa.cnf
#ChangeLog               bin                     openssl.exe
#EasyRSA-Start.bat       doc                     pki
#Licensing               easyrsa                 vars.example
#README-Windows.txt      libcrypto-3-x64.dll     x509-types

#EasyRSA Shell
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
# C:/Program Files/EasyRSA-3.1.0/pki/reqs/FreshTomato.req
# within the OpenVPN Server configuration the NoPass.key is being used
C:/Program Files/EasyRSA-3.1.0/pki/private/FreshTomatoNoPass.key
C:/Program Files/EasyRSA-3.1.0/pki/issued/FreshTomato.crt
C:/Program Files/EasyRSA-3.1.0/pki/dh.pem
```

## FreshTomato - Configuration - OpenVPN Server
VPN Tunneling -> OpenVPN Server ->

```shell
Certificate Authority Key - paste the ca.key, including -----BEGIN ENCRYPTED PRIVATE KEY----- and -----END ENCRYPTED PRIVATE KEY-----
Certificate Authority     - paste the ca.crt, including -----BEGIN CERTIFICATE----- and -----END CERTIFICATE-----
Server Certificate        - paste FreshTomato.crt, including -----BEGIN CERTIFICATE----- and -----END CERTIFICATE-----
Server Key                - paste FresTomatoNoPass.key, including -----BEGIN PRIVATE KEY----- and -----END PRIVATE KEY-----
Diffie Helman Parameters  - paste dh.pem, uncluding -----BEGIN DH PARAMETERS----- and -----END DH PARAMETERS-----
```

Save configuration. At this stage the NVRAM consumption should increase, in my case 61% of NVRAM is utilized.<br>
Press the *Start Now* button and within the Status section, you should be shown with new elements like, Current List, Routing Table, General Statistics.
In order to really proof that the tunnel has been set up, go to your SSH session which you established at the very beggining and execute

```shell
ifconfig
## among interfaces like br0, eth0, eth1, lo, vlan1, vlan2 you will also see tun21
tun21      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
           inet addr:10.0.6.241  P-t-P:10.0.6.241  Mask:255.255.255.240
           UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:0
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
## this will confirm that at least another virtual interface arose, and  it's IP comes from the VPN Subnet in VPN Tunneling Basic settings configuration pane.
```

## FreshTomato - Configuration - Generating Client Certificate and key

```shell
#EasyRSA Shell
./easyrsa build-client-full ovpn-Client1
#* Notice:
#Using Easy-RSA configuration from: C:/Program Files/EasyRSA-3.1.0/pki/vars

#* Notice:
#Using SSL: openssl OpenSSL 3.0.3 3 May 2022 (Library: OpenSSL 3.0.3 3 May 2022)

#.+...+......+++++++++++++....+
#+++++++++++++++++++++++*...++++++
#Enter PEM pass phrase: [CLIENT_PEM_PASSPHRASE]
#Verifying - Enter PEM pass phrase: [CLIENT_PEM_PASSPHRASE]
#-----
#* Notice:

#Keypair and certificate request completed. Your files are:
#req: C:/Program Files/EasyRSA-3.1.0/pki/reqs/ovpn-Client1.req
#key: C:/Program Files/EasyRSA-3.1.0/pki/private/ovpn-Client1.key

#Using configuration from C:/Program Files/EasyRSA-3.1.0/pki/safessl-easyrsa.cnf.init-tmp
#Enter pass phrase for C:/Program Files/EasyRSA-3.1.0/pki/private/ca.key: [CA_KEY_PASSPHRASE]
#Check that the request matches the signature
#Signature ok
#The Subject's Distinguished Name is as follows
#commonName            :ASN.1 12:'ovpn-Client1'
#Certificate is to be certified until Oct  3 22:10:18 2024 GMT (825 days)

#Write out database with 1 new entries
#Data Base Updated

#* Notice:
#Certificate created at: C:/Program Files/EasyRSA-3.1.0/pki/issued/ovpn-Client1.crt
#EasyRSA Shell
#
```

## Final thoughts
+ ca.crt is common for the clients, until it gets renewed on the OpenVPN server device
+ ovpn-ClientX.crt and ovpn-ClientX.key is generaged individually for each OpenVPN client and separatelly shared with anyone you'd like to share your VPN with
+ Following files are shared with each client (those ).
```shell
C:/Program Files/EasyRSA-3.1.0/pki/ca.crt #it's the same for each client
#those two goes in pair and are uniques for each client
C:/Program Files/EasyRSA-3.1.0/pki/issued/ovpn-Client1.crt
C:/Program Files/EasyRSA-3.1.0/pki/private/ovpn-Client1.key
```

## Links
+ [sekurak.pl](https://sekurak.pl/praktyczna-implementacja-sieci-vpn-na-przykladzie-openvpn/) - openVPN practical implementaion
+ [howtoforge.com](https://www.howtoforge.com/tutorial/how-to-install-openvpn-server-and-client-with-easy-rsa-3-on-centos-8/) - centos8, easyrsa 3, openvpn

## Summary
That's it.<br>
Tested on FreshTomato 2022.3, EasyRSA 3.1.0, OpenSSL 3.0.3, RT-N18U.<br>
Last update: 2022.07.01