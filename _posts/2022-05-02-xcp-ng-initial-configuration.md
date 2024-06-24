---
layout: post
title: "XCP-ng initial configuration"
permalink: "/xcp-ng-initial-configuration/"
subtitle: "XCP-ng left much to be desired, though it's a great solution for home lab"
cover-img: /assets/img/cover/img-cover-xen.jpg
thumbnail-img: /assets/img/thumb/img-thumb-xpng8.png
share-img: /assets/img/cover/img-cover-xen.jpg
tags: [HomeLab ,XCP-ng]
categories: [HomeLab ,XCP-ng]
---
The hypervisor of choice for this lab is XCP-ng.

## Configuration

The configurations looks l

## Option 1 - XOA - Install XenOrchestra appliance from the sources

Installation from the sources:

[xenorchestra.com](https://xen-orchestra.com/docs/installation.html#from-the-sources)
[@HomeTinyLab](https://www.youtube.com/watch?v=6RR3WtQIbe0)
[Github jarli01](https://github.com/Jarli01/xenorchestra_installer)

```shell
#execute on your debian or ubuntu
# the credentials after installation are:
# u: admin@admin.net
# p: admin
sudo bash
bash -c "$(curl https://raw.githubusercontent.com/Jarli01/xenorchestra_installer/master/xo_install.sh)"
```

Configure the self signed cetrtficate for the XOA appliance

```shell
# Self signed SSL
# Generate your key and cert from your XOCE installation
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:4096 -keyout /etc/ssl/private/key.pem -out /etc/ssl/certs/certificate.pem
# Now edit the xo-server.toml file
nano /opt/xen-orchestra/packages/xo-server/.xo-server.toml
# Comment-out or edit the port from 80 to 443 and add the cert and key to the appropriate locations within this file.
port = 443
cert = '/etc/ssl/certs/certificate.pem'
key = '/etc/ssl/private/key.pem'
# restart xo-serever.service
systemctl restart xo-server.service
```

Update the XOA is covered within this script [github Jarli01](https://github.com/Jarli01/xenorchestra_updater)

```shell
# execute the lines below to upgrade XOA
sudo bash
<password>
sudo curl https://raw.githubusercontent.com/Jarli01/xenorchestra_updater/master/xo-update.sh | bash

# in case of incorrect version of node is detected then you should execute
sudo n lts
# then execute again the first lines from this section
```

## Option 2 - XOACE - install XenOrchestra Community Edition

As mentioned above [ronivay](https://github.com/ronivay/XenOrchestraInstallerUpdater#image) provided the script which can be executed directly via SSH connection on the XCP-ng host.
Youtube video how to make it happen is available on the [@HomeTinyLab](https://www.youtube.com/watch?v=bzwFg6qrUoI) YouTube channel.

The approach described here, brings the version of XOA which does not drop the connectivity from the hypervisor, in 5s loops.

```shell
#run this directly on your XCP-ng host, when connected over SSH
sudo bash -c "$(curl -s https://raw.githubusercontent.com/ronivay/XenOrchestraInstallerUpdater/master/xo-vm-import.sh)"
# default username is xo with password admin
# ssh of the XOA is available with username xo with password xopass
# remember to change both passwords before putting the VM to acutal use
```

## Option 3 - XOACE - XenOrchestra Community Edition installation from sources

The prerequisite for this approach is that the VM's are already running, which means that you should have them provisioned alreay by making use of the prepackaged version of the XenOrchestra deploymedn, available within the XCP-ng installation, or via XE commands.

+ Link to the [documentation](https://xen-orchestra.com/docs/installation.html#from-the-sources)
+ Youtube tutorial from [@HomeTinyLab](https://www.youtube.com/watch?v=B6qX_nvd8Ac) - Christophe installs it sucesfully on debian 11.

In my case the main drawback with this method (on ubuntu 23.10 and node.js 21.X) comparing to the XenOrchestra Appliance bundled with the XCP-ng installation is that it loses the conectivity with the hypervisor in 5 loops.
The advantage is that the ISO import to the SR works, which was not available with the latter, based on my experience.

```shell
# the compilation will be conducted on the debian vm - version 23.10 (mantic)
sudo su #su - root
apt update
apt upgrade
apt install curl nfs-common neofetch
neofetch
# on 2023.10.24 - the 19.x has been deprecated
#curl -fsSl https://deb.nodesource.com/setup_19.x | bash -
# following the documentation - https://github.com/nodesource/distributions
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

NODE_MAJOR=21
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt-get update
sudo apt-get install nodejs -y

npm install --global yarn
# to verify the installed version of node
node -v
# install the packages needed by XenOrchestra
sudo apt install build-essential redis-server libpng-dev git python3-minimal libvhdi-utils lvm2 cifs-utils
# clone the code
git clone -b master https://github.com/vatesfr/xen-orchestra
# now it's time to build the appliance
yarn
#[5/5] Building fresh packages...
#$ husky install
#husky - Git hooks installed
# Done in 260.03s.
yarn build
#Tasks:    26 successful, 26 total
#Cached:    0 cached, 26 total
# Time:    2m41.142s
# Done in 161.96s.
#
# now the xenorchestra need to find the configuration file
# we are logged in as a root user
cd packages/xo-server
mkdir -p ~/.config/xo-server
cp sample.config.toml ~/.config/xo-server/config.toml 
# now start the xenorchestra
yarn start
# now open a web browser and navigate to the IP address of your
# VM where you have been compiling it
# you can login with default credentials
# username: admin@admin.net
# password: admin

# create a new user with admin rights, and remove the default one
# get back to the terminal and run CTRL + C to stop the xenorchestra

# make XOCE running as a service so it's start up automatically
# when we reboot the server

mkdir -p /etc/xo-server
cp sample.config.toml /etc/xo-server/config.toml
ls -lah /etc/xo-server/

yarn global add forever
yarn global add forever-service
# at this point we are still in /root/xen-orchestra/packages/xo-server folder
forever-service install orchestra -r root -s dist/cli.mjs
service orchestra status
# inactive
service orchestra start
```

## Patch XCP-ng

There is an option to patch the XCP-ng from the XOACE, once the system is patched it needs to be rebooted. To schedule it you can approach it the following way.

```shell
echo "/sbin/shutdown -r now" | at 19:00
at q
```

## Upgrade XCP-ng with yum

The video how to upgrade from 8.1 to 8.2 is available on [@HomeTinyLab](https://www.youtube.com/watch?v=jliok9FzwEc).
Update the master first if you have the pool, create a backup first before upgrading your XCP-ng host.
If you put the installation media into the host, it will ask you for the option of an upgrade as well.

```shell
# execute the code below on the XCP-ng terminal
yum update
export VER=8.2
wget https://updates.xcp-ng.org/8/xcp-ng-$VER.repo -O xcp-ng-$VER.repo
wget https://updates.xcp-ng.org/8/SHA256SUMS -O SHA256SUMS
wget https://updates.xcp-ng.org/8/SHA256SUMS.asc -O SHA256SUMS.asc
ls -lah
cp xcp-ng.8.2.repo /etc/yum.repos.d/xcp-ng.repo
# overwrite it
yum clean metadata
yum update
```

## Storage

If you pair xcp-ng with the Freenas, the on your dataset you create. [@HomeTinyLab](https://www.youtube.com/watch?v=bCPUlaUc6wc)

+ NFS (file level) - dataset, root, wheel, authorize only the xcp-ng host (by IP address), nfs v4 with options
+ iscsi (block level) - zvol (volume), device type, configured zvol as the extend, platform xen (thrue nas is optimizing it for the usecase), portal, no chap etc

Separate the network for the storage from the vm traffic, in case of pairing with Truenas, this can be done by making use of Mellanox cards, along with the increase of the MTU on that link.

## Spinning up VM's

To manage the XCP-ng you need to be familiar with xe commands, then you can do everything you need to do with through the ssh connection.

+ You can also make use of the [Xen Orchestra](https://xen-orchestra.com/docs/installation.html#xoa) web based interface which can be deployed once you connect over the browser to the IP address of your hypervisor - The official graphical client
+ Another option if you are in disposal of window box is to install the [XCP-ng center](https://github.com/xcp-ng/xenadmin/releases/), so called xenadmin

### debian - preparation

Debian distribution will be used as the place where the tooling will be made available, so the core services for XCP-ng available in Rocky won't be interrupted.

```shell
# show the version of the debian
lsb_release -a
# in this case it's 23.10 (mantic)
```


## Conclusions

Last update: 2022.05.10
