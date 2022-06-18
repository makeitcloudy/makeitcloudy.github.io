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

## Preparation - OpenSSL
### Linux
In case the OpenSSL is not available on your machine, but you can quickly spin up a VM, I've wrote a blog post [how to compile OpenSSL from sources](https://makeitcloudy.pl/how-to-compile-openssl-from-sources/).
### Windows
I'm sure there is a way to compile the OpenSSL on Windows (probably [this](https://github.com/openssl/openssl/blob/master/NOTES-WINDOWS.md) is the way to go), never the less there is a quicker way to make use of OpenSSL on this platform as well, especially if you do a bit of coding and make use of GIT already. OpenSSL exist in the following directory *C:\Program Files\Git\usr\bin* you may use it from there, or add it to *$env:PATH* and run it directly from the commandline. At the time of writing this I'm on

```bash
git --version
git version 2.36.1.windows.1
```
and it have the OpenSSL 1.1.1o
```bash
openssl.exe
OpenSSL> version
OpenSSL 1.1.1o  3 May 2022
```
## Howto
.
## Summary
I'm sure there are better ways doing it, but still it's a good starting point.<br>
It was tested on RouterBoard and CCR.<br>
Last update: 2022.04.15