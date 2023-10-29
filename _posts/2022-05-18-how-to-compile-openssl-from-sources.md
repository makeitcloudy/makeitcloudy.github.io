---
layout: post
title: "How to compile OpenSSL from sources"
permalink: "/how-to-compile-openssl-from-sources/"
subtitle: "OpenSSL 1.1.1 will receive updates until September 2023"
cover-img: /assets/img/cover/img-cover-padlock.jpg
thumbnail-img: /assets/img/thumb/img-thumb-padlock.jpg
share-img: /assets/img/cover/img-cover-padlock.jpg
tags: [HomeLab ,Certificates , SSL]
categories: [HomeLab ,Certificates , SSL]
---
Once the OpenSSL library is installed, you can make use of it, for preparing self signed certificates, chain certificates within each other, removing secrets from private keys, with the available API and all the blessing comming with that library. Sometimes it is just more convinient to perform it outside of the Mikrotik or NetScaler box, especially for one who is not doing this in regular basis.

## Prerequisites

+ preferably, machine with linux

## Background

+ Details about OpenSSL can be found on [github](https://github.com/openssl/openssl)
+ All releases can be found [here](https://www.openssl.org/source/old/)

## Howto

1. Install Rocky
2. Install Management tools

```bash
sudo mount /dev/cdrom /mnt
sudo bash /mnt/Linux/install.sh -d rhel -m 8
sudo umount /dev/cdrom
```

1. Compile OpenSSL from sources

```bash
# Install prereq packages and libraries
yum group install 'Development Tools'
yum install perl-core zlib-devel -y
#download openssl - at this point of time 1.1.1n
wget https://www.openssl.org/source/old/1.1.1/openssl-1.1.1n.tar.gz
tar -xf openssl-1.1.1n.tar.gz
cd openssl-1.1.1n
openssl version -a
# 1.1.1k in the system
# now the existing version is replaced by the one downloaded
./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
make
make test
# wait until the compilation process ends
# once completed, install OpenSSL
make install
# configure shared libraries for OpenSSL
cd /etc/ld.so.conf.d/
nano openssl-1.1.1n.conf
# paste the openssl library path directory
/usr/local/ssl/lib
# reload dynamic link
ldconfig -v
# configure openSSL binary, to have it linked with the version compiled
mv /bin/openssl /bin/openssl.org
# create new environment for OpenSSL
nano /etc/profile.d/openssl.sh
# paste following content
#Set OPENSSL_PATH
OPENSSL_PATH="/usr/local/ssl/bin"
export OPENSSL_PATH
PATH=$PATH:$OPENSSL_PATH
export PATH
# save and exit
# add execute permissions to openssl.sh
chmod +x /etc/profile.d/openssl.sh
# load OpenSSL environment and check the PATH bin directory
source /etc/profile.d/openssh.sh
echo $PATH
which openssl
# should result as /usr/local/ssl/bin/openssl
# it would mean thatn OpenSSL on CentOS has been updated
openssl version
# should result with 1.1.1n
```

## Summary

It may be far from being perfect, never the less good enough for a home lab.
Tested on Cento8 Stream. OpenSSL 1.1.1n.
Last update: 2022.05.18