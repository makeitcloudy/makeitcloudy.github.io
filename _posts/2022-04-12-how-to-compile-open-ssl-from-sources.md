---
layout: post
title: "How to compile open ssl from sources"
permalink: "/how-to-compile-open-ssl-from-sources/"
subtitle: "OpenSSL will bring some grip, here and there"
cover-img: /assets/img/how-to-compile-open-ssl-from-sources/img-cover.jpg
thumbnail-img: /assets/img/how-to-compile-open-ssl-from-sources/img-thumb.jpg
share-img: /assets/img/how-to-compile-open-ssl-from-sources/img-cover.jpg
tags: [HomeLab ,Certificates ,SSL]
categories: [HomeLab ,Certificates ,SSL]
---
*As of 2022.04.12 - draft*
Once the OpenSSL library is installed, you can make use of it, for preparing self signed certificates, chain certificates within each other, removing secrets from private keys, with the available API and all the blessing comming with that library. Sometimes it is just more convinient to perform it outside of the Mikrotik or NetScaler box, especially for one who is not doing this in regular basis.

## Prerequisites
+ preferably, machine with linux

## Howto
+ Details about OpenSSL can be found on [github](https://github.com/openssl/openssl)
+ All releases can be found [here](https://www.openssl.org/source/old/)
+ Some useful script can be found on [github](https://gist.github.com/HQJaTu/963db9af49d789d074ab63f52061a951)

```bash
#here it is done on debian
sudo apt update
sudo apt upgrade
openssl help
openssl version
sudo apt install make
sudo apt install gcc
sudo apt install zlib
sudo apt-get install libz-dev
sudo apt-get install lib1g-dev
sudo apt-get install zlib1g-dev
sudo apt install build-essential checkinstall zlib1g-dev -y
cd /usr/local/src/
sudo wget https://www.openssl.org/source/old/1.1.1/openssl-1.1.1n.tar.gz
sudo chmod +x openssl-1.1.1n.tar.gz
sudo tar -xf openssl-1.1.1n.tar.gz
cd openssl-1.1.1n/
openssl version -a
# --prefix and --openssldir = Set the output path of the OpenSSL.
# shared = force to create a shared library.
# zlib = enable the compression using zlib library.
sudo ./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
sudo make
sudo make test
sudo make install
cd /etc/ld.so.conf.d/
sudo nano openssl-1.1.1n.conf
# paste the following content into the file: /usr/local/ssl/lib
# save and exit
sudo ldconfig -v
# configure OpenSSL library
sudo mv /usr/bin/c_rehash /usr/bin/c_rehash.bkp
sudo mv /usr/bin/openssl /usr/bin/openssl.bkp
sudo nano /etc/environment
# add new OpenSSL binary directory as below
# PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/ssl/bin"
# reload the environment file and test the $PATH
source /etc/environment
echo $PATH
# check again the OpenSSL file
which openssl
# at this point you should see the downloaded version of the OpenSSL binary
openssl version
```

## Summary
It may be far from being perfect, please use this only for a home lab.<br>
Last update: 2022.04.12