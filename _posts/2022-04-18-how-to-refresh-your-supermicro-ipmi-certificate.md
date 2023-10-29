---
layout: post
title: "Supermicro IPMI self signed certificates"
permalink: "/how-to-refresh-your-supermicro-ipmi-certificate/"
subtitle: "OpenSSL will bring some grip, here and there"
cover-img: /assets/img/cover/img-cover-padlock.jpg
thumbnail-img: /assets/img/thumb/img-thumb-padlock.jpg
share-img: /assets/img/cover/img-cover-padlock.jpg
tags: [HomeLab ,Certificates ,SSL ,IPMI ,Supermicro]
categories: [HomeLab ,Certificates ,SSL ,IPMI ,Supermicro]
---
*As of 2022.04.16 - draft*

Self signed certificates on supermicro differs between the SM IPMI releases, by format, and the key lenght.

## Background

+ Some useful script can be found on [github](https://gist.github.com/HQJaTu/963db9af49d789d074ab63f52061a951)
+ Some useful commands can be found on [github](https://gist.github.com/makeitcloudy/e0040f9390b2cf458e95227439cd931c)

## Refresh Superimcro IPMI certificate - newer firmware - motherboard of X9 series

This was tested on 3.48 version of the firmware.
If you have a newer firmware than 3.19, the you can use following method to generate the self signed certificate, which then can be used to secure your IPMI connection.

The node1ipmi-Config.cfg file contains the following

```shell
[ CA_default ]
 default_md    = sha256

[ req ]
 default_bits = 2048
 distinguished_name = req_distinguished_name
 encrypt_key = no
 prompt = no
 string_mask = nombstr
 req_extensions = v3_req

[ v3_req ]
 basicConstraints = CA:FALSE
 keyUsage = digitalSignature, keyEncipherment, dataEncipherment
 extendedKeyUsage = serverAuth
 subjectAltName = @alt_names

[alt_names]
 DNS.1 = node1ipmi
 DNS.2 = node1ipmi.lab

[ req_distinguished_name ]
 countryName = PL
 stateOrProvinceName = MarshLand
 localityName = MarshLand
 0.organizationName = HomeLab
 organizationalUnitName = IT Services
 commonName = node1ipmi.lab
 emailAddress = EmailAddress@domain.pl
```

Now when the configuration file is in place, execute the following

```shell
cd ~
openssl genrsa 2048 > node1ipmi.key
#openssl req -out node1ipmi-CertificateSigningRequest.csr -key node1ipmi-Key.key -new -config node1ipmi.cfg
openssl req -out node1ipmi.csr -key node1ipmi-Key.key -new -config node1ipmi-Config.cfg
```

## Second option

```shell
openssl req -x509 -newkey rsa:2048 -sha256 -days 1095 -nodes -keyout node1ipmi.key -out node1ipmi.crt -subj "/CN=node1ipmi.lab" -addext "subjectAltName=DNS:ipmi1.lab,DNS:https://node1ipmi.lab,IP:IpAddressOfIPMI"
openssl pkcs12 -export -out node1ipmi.pfx -inkey node1ipmi.key -in node1ipmi.crt
openssl pkcs12 -export -out node1ipmi.p12 -inkey node1ipmi.key -in node1ipmi.crt -certfile node1ipmi.crt
```

Then on the web interface of your ipmi, choose Configuration -> SSL Certification and pick the .csr

## Refresh Supermicro IPMI certificate - older firmware - motherboard of X9 series

For older firmware for the motherboard which was longer out of further support, but are still great use for the homelab.

```shell
openssl req -x509 -newkey rsa:1024 -sha1 -days 1095 -nodes -keyout node3ipmiKey.pem -out node3ipmiCert.pem -subj "/CN=node3ipmi.lab" -addext "subjectAltName=DNS:ipmi3.lab,DNS:https://node3ipmi.lab,IP:IpAddressOfYourIPMI"
openssl pkcs12 -export -out node3ipmi.p12 -inkey node3ipmiKey.key -in node3ipmiCert.pem -certfile node3ipmiCert.pem
openssl pkcs12 -export -out node3ipmi.p12 -in node3ipmiCert.pem -certfile node3ipmiCert.pem
openssl pkcs12 -export -out node3ipmi.p12 -inkey node3ipmiKey.key -certfile node3ipmiCert.pem
openssl pkcs12 -export -nodes -out bundle.pfx -inkey mykey.key -in certificate.crt -certfile ca-cert.crt     -passout pass:
openssl pkcs12 -export -nodes -out node3ipmi.p12 -inkey node3ipmiKey.pem -in node3ipmiCert.pem -passout pass:
openssl pkcs12 -export -out node3ipmi.pfx -inkey node3ipmiKey.pem -in node3ipmiCert.pem
```

## Summary

For the IPMI certificate - Something is missing, needs some further work to validate the config which was done back then.

Last update: 2022.04.18
