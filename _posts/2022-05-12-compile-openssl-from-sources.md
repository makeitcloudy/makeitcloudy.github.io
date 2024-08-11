---
full-width: true
layout: post
title: "Compile OpenSSL from sources"
permalink: "/compile-openssl-from-sources/"
subtitle: "OpenSSL 3.0.9 FIPS and 1.1.1"
cover-img: /assets/img/cover/img-cover-padlock.jpg
thumbnail-img: /assets/img/thumb/img-thumb-padlock.jpg
share-img: /assets/img/cover/img-cover-padlock.jpg
tags: [HomeLab, Rocky, Debian, Certificates, SSL]
categories: [HomeLab, Rocky, Debian, Certificates, SSL]
---
OpenSSL library is used for:

* self signed certificates, chainining certificates within each other, removing secrets from private keys, etc
* it is just more convinient to perform it outside of the Mikrotik or NetScaler box, especially for one who is not doing this in regular basis

## Background

### Debian 12

* youtube - [OpenSSL FIPS Provider](https://www.youtube.com/watch?v=geAtEXbHaFg)
* github - [OpenSSL FIPS support](https://github.com/openssl/openssl/blob/openssl-3.2/README-FIPS.md)

### Rocky8

* Details about OpenSSL can be found on [github](https://github.com/openssl/openssl)
* All releases can be found [here](https://www.openssl.org/source/old/)

## Prerequisites

* preferably, machine with linux

1. Install Debian / Rocky
2. Install VM tools

```shell
# Rocky contains an alternative way of installing the management tools
# the method below works for the CentOS 8 Stream which is not supported anymore
sudo mount /dev/cdrom /mnt
sudo bash /mnt/Linux/install.sh -d rhel -m 8
sudo umount /dev/cdrom
```

## Debian 12 - OpenSSL

```shell
# login as sudoUser

# install prerequisites
sudo apt install build-essential

# https://youtu.be/geAtEXbHaFg?t=687
mkdir -p sources
cd sources
wget https://www.openssl.org/source/openssl-3.0.9.tar.gz
tar -xf openssl-3.0.9.tar.gz 
cd openssl-3.0.9/
./Configure enable-fips

make
make test
sudo make install

openssl version

### at this stage the openssl is ready, further configuration is with regrads with the FIPS compliance

### Configuring the FIPS provider
openssl version -d
# Locate the openssl.cnf file in the OEPNSSLDIR directory and load it into an editor
# find the line that includes the fipsmodule.cnf file, uncomment it, and replace the filename with the full absolute path to fipsmodule.cnf

.include /usr/local/ssl/fipsmodule.cnf

# do not use a relative filename
# find the line specifying the fips section and uncomment it
fips = fips_sect

# next load the fipsmodule.ndf file (in t he same directory) into an editor
# for now we are goind to stop the FIPS provider from activating itself by default
# this way we can use the FIPS provider if we want to, but we can also use OpenSSL without FIPS
# comment out the "activate" line:

# activate = 1

# Note that this is NOT sufficient to set activate to 0

# Check that everything worked

openssl list -providers -provider fips -provider base

# If we don't explicitly load the fips/base providers then we should get the default provider

openssl list -providers

# https://github.com/openssl/openssl/blob/openssl-3.2/README-FIPS.md


### Make everything use FIPS
# https://youtu.be/geAtEXbHaFg?t=1497
```

## Rocky 8 - Open SSL

Compile OpenSSL from sources

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

It may be far from being perfect, though good enough for a home lab.
Tested on Debian 12.5, Cento8 Stream. OpenSSL 3.0.9 FIPS and 1.1.1n.

Last update: 2024.06.25
