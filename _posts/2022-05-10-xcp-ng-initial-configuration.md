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
There are at least few hypervisors on the market which would be a good fit, though in this case as it is Citrix oriented lab, the XCP-ng was the choice. Hyper-V server core, would do the trick, esxi as well, though both of them will be missing the integration of the SDK.
Xen Orchestra is included within the package, though later on, once you do the initial configuraiton, you can scrap the appliance and compile one from sources, or use the script provided by [ronivay](https://github.com/ronivay/XenOrchestraInstallerUpdater).
The first contact with xcp-ng may be harsh, when you don't have stron RedHat background, though the convinience will come with time.

## Configuration

Once the XCP-ng is installed, login to it's web interface via web browser and deploy the XenOrchestra comming within the installation package. It should be available for the next 30days after the installation, which is enough amount of time, to perform the initial configuration of your hypervisor, along with spinning up debian or rocky linux (RHEL) to compile the Xen Orchestra Community Ediction from sources, which does not have the functioanlity limitations after the 30 days trial period.

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

### rocky - preparation

It seems this can be done in many more sofisticated ways, never the less this approach works for the home lab usecase. Rocky is a continuation of the CentOS stream.

#### rocky - XCP-ng tools installation

For some reason the default way of mounting the xpn-ng tools and installation with the included script does not work. Thankfully [Sam Morreel](https://www.saasycloud.com/solutions/blog/install-xcp-ng-guest-tools-on-rocky-linux-9) shared how to deal with it and it actually works.

```shell
#Install XCP-NG guest tools on Rocky Linux 9
#November 21st at 6:55pm /  XCP-NG, ROCKY LINUX, XEN GUEST AGENT

#Author: Sam Morreel. Posted on LinkedIn

#Update 2022-10-05: Upstream XCP-NG has released patches and it includes xentools updates to 
#support Rocky 9, Alma 9, CentOS 9 etc. To revert the manual install:
rm -f /usr/share/oem/xs/xe-linux-distribution
rm -f /usr/share/oem/xs/xe-daemon
rm -f /etc/systemd/system/xe-linux-distribution.service
rmdir /usr/share/oem/xs
#Then, mount the tools ISO as normal and run the install.sh script and your OS will automatically be detected.

#Orginal Article:

# Currently as of this writing, there is no automatic way to install Linux guest tools using 
# XCP-NG's guest-tools.iso or EPEL on Rocky Linux 9, CentOS 9/stream. 
# The packages (specifically xe-guest-utilities-latest) are simply not yet available in their respective repos, 
# and the guest-tools.iso won't automatically install them.
# Fear not, there's a few simple steps to get this working. 
# Everything you need is located on the guest-tools.iso file. 
# The base install for this article uses the Rocky Linux mininal install.
# All commands are executed as root, or alternatively sudo.
# First, set the DVD drive to load the guest-tools.iso
# Then, mount the .iso:

mount /dev/sr0 -t iso9660 /mnt

# Next, we will need to create directory structure that doesn't likely exist on the minimal install. 
# This matches what the xe-linux-distribution.service file expects:

mkdir -p /usr/share/oem/xs

# Now let's copy over the required files.
# The distro check bash script:

cp /mnt/Linux/xe-linux-distribution /usr/share/oem/xs

# The guest agent daemon (ensure the file is executable):

cp /mnt/Linux/xe-daemon /usr/share/oem/xs

# Finally, the service file:

cp /mnt/Linux/xe-linux-distribution.service /etc/systemd/system

# Now start the service:

systemctl start xe-linux-distribution

# Check the status:

systemctl status xe-linux-distribution

#     Loaded: loaded (/etc/systemd/system/xe-linux-distribution.service; enabled; vendor preset: disabled)
#     Active: active (running) since Sun 2022-08-14 15:40:01 CST; 55min ago
#    Process: 1297 ExecStartPre=/usr/share/oem/xs/xe-linux-distribution /var/cache/xe-linux-distribution (code=exited, status=0/SUCCESS)
#   Main PID: 1302 (xe-daemon)
#      Tasks: 12 (limit: 23456)
#     Memory: 9.9M
#        CPU: 4.613s
#     CGroup: /system.slice/xe-linux-distribution.service
#             â”œâ”€1302 /usr/share/oem/xs/xe-daemon
#             â””â”€1307 logger -t xe-daemon -p debug

#Aug 14 15:40:01 localhost.localdomain systemd[1]: Starting Linux Guest Agent...
#Aug 14 15:40:01 localhost.localdomain systemd[1]: Started Linux Guest Agent.
#If you get the output above, you're good to go. To persist the service through reboots:

systemctl enable xe-linux-distribution

#That's a wrap! Note I have only tested this on Rocky Linux 9 and your mileage may vary on other OSs like CentOS 9 / Stream 9 etc. but this procedure should work for those as well.

#Enjoy!
```

### rocky - add a volume

The initial step after the installation of the xcp-ng tools is adding a new volume, which will be mainly used as the Storage Repository of the ISO files.

```shell
ls /dev/xvd*
fdisk -l
fdisk /dev/xvdb
# here you get into the partitioning of the volume you added to the VM
# Command: n
# Command: 1
# Command: w

# create a filesystem
mkfs.ext4 -f /dev/xvdb

#mount the filesystem
mount /dev/xvdb /data

# /ect/fstab contains following entry
/dev/xvdb       /data   ext4    defaults     0   0
```

Now once the volume is mounted it's time to configure the NFS.

#### rocky - NFS setup

Configuration of the NFS. NFS ISO Storage Repository is being used by the XCP-ng.

```shell
# execute those under the context of the user who is in the sudoers group
dnf install nfs-utils -y
systemctl start nfs-server.service
systemctl enable nfs-server.service
systemctl status nfs-server.service
nano /etc/exports
```

/etc/exports file content:

```shell
# {network address} - is the address of the network from which the NFS should be reachable
# it should contain the network range of the management interface of your XCP-ng
# the plan is that the SR NFS ISO repository will be used
/data/nfs_share/labIso/ {network address}/24(rw,sync,no_all_squash,root_squash)
/data/nfs_share/labIso/ {networka ddress}/24(rw,sync,no_all_squash,root_squash)
```

Continuation of the NFS configuration

```shell
exportfs -arv
exportfs -s
firewall-cmd --permanent --add-service=nfs
# not sure about those need to check it later
firewall-cmd --reload
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent -add-service=mountd
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```

#### rocky - SAMBA setup

Configuration of the Samba. SMB is the protocol which allows here uploading the files to the mount points of Samba and NFS. Rocky has the extra disk which is formated as ext4 and mounted as /data on the filesystem.

```shell

groupadd labusers

mkdir /data/smb_share
mkdir /data/smb_share/labIso

mkdir -p data/smb_share/public
mkdir -p /data/nfs_share/labIso
chown -R nobody:nobody /data/smb_share/
  
nano /etc/samba/smb.conf
```

/etc/samba/smb.conf content:

```shell
[global]
workgroup = WORKGROUP
dos charset = cp850
unix charset = ISO-8859-1

log level = 2
dns proxy = no
map to guest = Bad User
server string = Samba Server %v
netbios name = SAMBA-SERVER
security = USER
disable spoolss = yes
min protocol = SMB2
wins support  No
ntlm auth = true
idmap config * : backend = tbd

[labIso-smb]
comment = smb share labIso-smb
inherit acls = Yes
path =  /data/smb_share/labIso
valid users = @labusers root
browsable =yes
writable = yes
guest ok = no
read only = no

[labIso-nfs]
comment = smb share labIso-nfs
inherit acls = Yes
path = /data/nfs_share/labIso
valid users = @labusers root
guest ok = no
writable = yes
browsable = yes
```

further samba related configuration settings

```shell
chmod -R 0755 /data/smb_share/   
chmod -R 0770 /data/smb_share/labIso/
chmod -R 0770 /data/nfs_share/labIso/
chown -R root:labusers /data/smb_share/labIso/
chown -R root:labusers /data/nfs_share/labIso/
chmod -R o+rx /data/nfs_share/labIso/

smbpasswd -a username
useradd -g labusers username
usermod -G labusers username

systemctl restart smb nmb
testparm
```

At this point, you should be able to reach the abovementioned pahts over SMB, and create the SR NFS ISO location on the XCP-ng host over NFS v 3.0. It's time to upload the ISO files, which can not be downloaded directly from the internet, or you have them customized already.

## Conclusions

Last update: 2022.05.10
