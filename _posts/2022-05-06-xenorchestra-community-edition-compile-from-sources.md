---
layout: post
title: "Xen Orchestra Community Edition - compile from sources"
permalink: "/xenorchestra-community-edition-compile-from-sources/"
subtitle: "XO Community edition to orchestra your XCP-ng on debian 12.5"
cover-img: /assets/img/cover/img-cover-xen.jpg
thumbnail-img: /assets/img/thumb/img-thumb-xpng8.png
share-img: /assets/img/cover/img-cover-xen.jpg
tags: [HomeLab ,Debian ,XCP-ng]
categories: [HomeLab ,Debian ,XCP-ng]
---
The hypervisor of choice for this lab is XCP-ng.

## Links

* [Xen Orchestra - installation from sources](https://xen-orchestra.com/docs/installation.html#from-the-sources)

### Storage

If you pair xcp-ng with the Freenas, then on your dataset you create separate the network for the storage from the vm traffic, in case of pairing with Truenas, this can be done by making use of Mellanox cards, along with the increase of the MTU on that link. [@HomeTinyLab](https://www.youtube.com/watch?v=bCPUlaUc6wc)

+ NFS (file level) - dataset, root, wheel, authorize only the xcp-ng host (by IP address), nfs v4 with options
+ iscsi (block level) - zvol (volume), device type, configured zvol as the extend, platform xen (thrue nas is optimizing it for the usecase), portal, no chap etc

### Spinning up VM's

To manage the XCP-ng you need to be familiar with xe commands.

+ You can also make use of the [Xen Orchestra](https://xen-orchestra.com/docs/installation.html#xoa) web based interface which can be deployed once you connect over the browser to the IP address of your hypervisor - The official graphical client
+ Another option if you are in disposal of window box is to install the [XCP-ng center](https://github.com/xcp-ng/xenadmin/releases/), so called xenadmin - though this option requires a windows machine

## Assumptions

0. bash scripts: copied to /opt/scripts/ on XCP-ng
1. XCP-ng tools: /opt/xensource/packages/iso

2. Local iso repository: /var/opt/xen/ISO_Store
3. Local iso repository: [debian netinst cd image](https://www.debian.org/CD/netinst/) in (-rw-r--r--)

4. XCP-ng: /var/opt/xen/ISO_Store - contains the Debian 12.5 ISO
5. XCP-ng: /opt/scripts - contains the vm_create_bios.sh script
6. XCP-ng: /opt/scripts - contains the vm_add_disk.sh script

## 1. Debian - Installation

Run the code below to create the debian VM:

```shell
/opt/scripts/vm_create_bios.sh --VmName 'debian-XO' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 0 --TemplateName 'Debian Bookworm 12' --IsoName 'debian-12.5.0-amd64-netinst.iso' --IsoSRName 'hdd_LocalISO' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:99:52' --StorageName 'node4_ssd_sdd' --VmDescription 'node4_debian12_xo'
```

Proceed with the following steps to complete the installation:

```code
# in Xen Orchestra
1. Install Debian
2. in the gui based installations
2.1 (software selection step - untick the graphical user env)
2.2 (software selection step - tick SSH server, standard system utilities)
3. unmount installation media
4. reboot VM
5. login to the VM as regular user
6. ip a
7. close the terminal and switch to regular SSH connection
```

## 2. Debian - Initial Configuration

Login via SSH to the VM with the regular user

```shell
echo $PATH
# edit root user's .bashrc file
export PATH=$PATH:/sbin:/usr/sbin
sudo nano /root/.bashrc
# add the following line at the end of the file
export PATH=$PATH:/sbin:/usr/sbin

apt install sudo
usermod -aG sudo sudoUser

nano /etc/sudoers
# modify the User privilege specification
sudoUser ALL=(ALL:ALL) ALL
```

8. insert guest-tools.iso to VM, mount and install

Login as the sudoUser - the one which has been created as regular user during the installation of the system.

```shell
# mount the guest-tools.iso file - which is updated during the updates of the xcp-ng
sudo mount /dev/cdrom /mnt
cp /mnt/Linux/xe-guest-utilities_7.30.0-12_amd64.deb ~
sudo umount /dev/cdrom
cd ~
sudo dpkg -i xe-guest-utilities_7.30.0-12_amd64.deb
rm -f xe-guest-utilities_7.30.0-12_amd64.deb

#unmount guest-tools
sudo unmount /dev/cdrom

#  update OS
sudo apt update
sudo apt upgrade
```

9. eject guest-tools.iso from VM

## 3. Debian - Configuration

## 3.1 XO Community Edition - Installation

[Install XenOrchestra Community Edition from Sources](https://xen-orchestra.com/docs/installation.html#from-the-sources)

```shell
# the configuration has been tested on 2024.06.24
# debian 12.5 bookworm | node 12.15.3 LTS | npm 10.7.0

# Static hostname: debianXO
#  Virtualization: xen
# Operating System: Debian GNU/Linux 12 (bookworm)  
#          Kernel: Linux 6.1.0-21-amd64


# run the commands with root account
apt install curl -y

# install node (pick latest LTS for XO)

# installs nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
# install node 20.15.3 LTS

# restart terminal - nvm will arise then

# download and install Node.js (you may need to restart the terminal)
nvm install 20
# verifies the right Node.js version is in the environment
node -v # should print `v20.15.0`
# verifies the right NPM version is in the environment
npm -v # should print `10.7.0`

# Install yarn
npm install --global yarn
#npm install -g npm@10.8.1

# Install packages
apt-get install build-essential redis-server libpng-dev git python3-minimal libvhdi-utils lvm2 cifs-utils nfs-common ntfs-3g -y

# Make sure redis is running
systemctl restart redis.service
# Ensure it's working
redis-cli ping
# should receive PONG

# Fetching code
cd /opt
sudo git clone -b master https://github.com/vatesfr/xen-orchestra

cd /opt/xen-orchestra
yarn
yarn build
#this process will take roughly few minutes

###
mkdir -p /etc/xo-server
cp /opt/xen-orchestra/packages/xo-server/sample.config.toml /etc/xo-server/config.toml

yarn global add forever
yarn global add forever-service
cd /opt/xen-orchestra/packages/xo-server
forever-service install orchestra -r root -s dist/cli.mjs

###
service orchestra start
service orchestra status
```

## 3.2 XO Community Edition - Upgrade

Follow the steps below to update XO:

```shell
cd /opt/xen-orchestra/
git checkout .
git pull --ff-only
yarn
yarn build
service orchestra restart
service orchestra status
```

## Summary

Tested on:
* Debian 12.5
* XCP-ng 8.2.1

Last update: 2024.06.24
