---
layout: post
title: "How to configure Mikrotik DDNS"
permalink: "/how-to-configure-mikrotik-DDNS/"
subtitle: "External DDNS provider, not the Mikrotik cloud approach"
cover-img: /assets/img/cover/img-cover-mikrotik.jpg
thumbnail-img: /assets/img/thumb/img-thumb-mikrotik.png
share-img: /assets/img/cover/img-cover-mikrotik.jpg
tags: [HomeLab ,Networking ,DDNS, Mikrotik]
categories: [HomeLab ,Networking ,DDNS, Mikrotik]
---
There are few ways which this can be achieved. The simplest way is to use Mikrotik Cloud, by enabling one *IP->Cloud->DDNS enabled* within the Winbox GUI. Another option is to reach out towards the third party provider with the use of curl, and update their systems with the information of Mikrotik WAN IP address. Another option is that you have some device behind your NAT, which is doing that for you like freshTomato, or raspberryPI or whatever else.

## Prerequisites

+ Account on one of DDNS providers, here [afraid freedns](https://freedns.afraid.org/) is being used.
+ Mikrotik device connected to the internet

## Howto

As it was mentioned within the fist paragraph there are few options if you have a dynamic IP address interface, and your goal is to expose some services towards the internet with making use of a domain name.

1. Mikrotik Cloud. Your device is assigned within a subdomain within the *sn.mynetname.net* domain which contains a bunch of generated characters. If this is okay, then that's fine and good enough. (What's more you will be informed whether the device is behind NAT or not)<br>
It looks that Mikrotik cloud is being used to update the time on your device, for instance if you have not configured the NTP client on your device.

```shell
 ## this command disables the Mikrotik cloud based time update
 ## remember to configure your ntp client, otherwise your OpenVPN configuration may not work
 ## as well as you may experience other quirks
 /ip/cloud/set update-time=no

## configure the NTP client
 /system ntp client
set enabled=yes
/system ntp client servers
add address=0.pl.pool.ntp.org
add address=1.pl.pool.ntp.org
add address=2.pl.pool.ntp.org

## if it is succesfull it looks like this
/system/ntp/client> /system/ntp/client/print 
         enabled: yes
            mode: unicast
         servers: 0.pl.pool.ntp.org,1.pl.pool.ntp.org,2.pl.pool.ntp.org
      freq-drift: 0 PPM
          status: synchronized
   synced-server: 1.pl.pool.ntp.org
  synced-stratum: 2
   system-offset: 0.907 ms
```

If you are in disposal of your own domain, then you may create the pointer which will be redirecting your domain or a subdomain towards that randomized number assigned to you from Mikrotik cloud. In case you are not having one, read along

2. Afraid DNS, along with other DDNS providers allow you creating free accounts, and creating subdomains, within the list of domains which they expose. Once you create one, you can get the [direct URL](https://freedns.afraid.org/dynamic/) for your subdomain.<br>
FreeDNS from afraid exposes https://freedns.afraid.org/dynamic/update.php?__UNIQUE_STRING__ which you need to pull in order to update the IP entry within their mapping. In order to do that execute following command

```shell
#[MIKROTIK_ADMIN_USER] is the name of your user which have full permissions to manage the Mikrotik device
#[__UNIQUE_STRING__] is the string you get from the https://freedns.afraid.org/dynamic/ Dynamic URL weblink, it contains at least 32 alphanumeric characters

/system script
add dont-require-permissions=no name=ddns_afraiddns owner=[MIKROTIK_ADMIN_USER] policy=read,write,test source="/tool fetch mode=https url=\"https://freedns.afraid.org\" http-data=\"dynamic/update.php\\\?[__UNIQUE_STRING__]\" keep-result=no"
```

Once this is done the script should be scheduled.<br>
If your IP address changes frequently then you may consider lowering the interval in this case, 36h is more than enough.

```shell
/system scheduler
add interval=1d12h name=schedule_ddns_afraiddns policy=read,write,test start-date=apr/14/2022 start-time=14:30:30
```

Once this is done, your dynamic IP address should be bound with the desired domain.

## Summary

Tested on ROS 7.3.1.

That's it.

Last update: 2022.04.14
