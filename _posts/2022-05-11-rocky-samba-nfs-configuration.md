---
layout: post
title: "Rocky 9.4 Samba and NFS configuration"
permalink: "/rocky-samba-nfs-configuration/"
subtitle: "Rocky 9.4 Samba na NFS setup - ISO repository for XCP-ng"
cover-img: /assets/img/cover/img-cover-linux-rocky.jpg
thumbnail-img: /assets/img/thumb/img-thumb-rocky.png
share-img: /assets/img/cover/img-cover-linux-rocky.jpg
tags: [HomeLab , Rocky, XCP-ng]
categories: [HomeLab , Rocky, XCP-ng]
---

Here are the steps how to set up Rocky 9.4 as Samba and NFS share for XCP-ng and it's VMs.

## Links

* [How To Create a New Sudo-enabled User on Rocky Linux 8 [Quickstart]](https://www.digitalocean.com/community/tutorials/how-to-create-a-new-sudo-enabled-user-on-rocky-linux-8-quickstart)
* [How to Install Samba Server on Rocky Linux 9 / AlmaLinux 9](https://www.linuxbuzz.com/install-samba-server-on-rockylinux-almalinux/)

## 0. Assumptions

0. bash scripts:

* copied to /opt/scripts/ on XCP-ng

1. XCP-ng tools:

* XCP-ng tools: /opt/xensource/packages/iso

2. ISO: 

* local iso repository - /var/opt/xen/ISO_Store
* [rocky minimal iso](https://rockylinux.org/download) in XCP-ng local iso repository (-rw-r--r--)

3. XCP-ng:

* /var/opt/xen/ISO_Store - contains the Rocky9 ISOroot
* /opt/scripts - contains the vm_create_bios.sh script
* /opt/scripts - contains the vm_add_disk.sh script

```bash
# run code in the XCP-ng terminal
/opt/scripts/vm_create_bios.sh --VmName 'rockyFS' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 0 --TemplateName 'Rocky Linux 9' --IsoName 'Rocky-9.4-x86_64-minimal.iso' --IsoSRName 'hdd_LocalISO' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:99:50' --StorageName 'node4_ssd_sdd' --VmDescription 'node4_rocky9_nfs_smb'
```

## 1. Installation

Proceed the following steps to complete the installation:

```code
# in Xen Orchestra
1. Install Rocky Linux
2. Set Root password
3. Installation Destination -> pick the base disk
3.1 Begin Installation
3.2 Reboot system

# in XO virtual Console 
4. login to the VM by making use of root account
```

### 2. Initial Configuration

Run the initial configuration commands via XenOrchestra virtual terminal. At this point by default you won't be able to login via ssh to the VM.

```bash
# rename the hostname
hostnamectl
hostname rocky9FS
# add extra user with sudo permissions
adduser sudoUser
passwd sudoUser
usermod -aG wheel sudoUser
su - sudoUser
ip a
exit
exit
```

At this point leave the Xen Orchestra Virtual Terminal and login via ssh to the VM, by making use of the sudoUser account.

5. eject the installation media
6. mount *guest-tools.iso* to rocky9 vm in XO

```bash
su
# provide root password
# install XCP-ng tools
blkid
mkdir -p /media/cdrom
mount /dev/sr0 /media/cdrom
cd /media/cdrom/Linux
# install xcp-ng tools
bash install.sh -d rhel -m 9
cd ~
umount /dev/sr0 /media/cdrom
```

7. eject *guest-tools.iso* from rocky9 VM in XO

```bash
# update
yum update
```

### 3. Add drive and volume

In XCP-ng terminal, run code

```bash
# add extra disk - dedicated for NFS and SMB storage
/opt/scripts/vm_add_disk.sh --vmName "rockyFS" --storageName "node4_hdd_sdc_lsi" --diskName "rockyFS_dataDisk" --deviceId 4 --diskGB 160  --description "rockyFS_filer_nfs_smb"
```

In Rocky9 VM, run code

```bash
mkdir -p /data
ls -lah /dev/xvd*
sudo fdisk -l
# nothice which xvd[x] is the drive you have just added which is 160GB
sudo fdisk /dev/xvde
# here you get into the partitioning of the volume you added to the VM
# Command: n
# Command: p
# Command: 1
# Enter
# Enter
# Command: w

# create a filesystem
mkfs -t ext4 /dev/xvde

#mount the filesystem
mount /dev/xvde /data

vi /etc/fstab
# INSERT
# /ect/fstab contains following entry
/dev/xvde   /data   ext4    defaults    0   0
# ESC
# :wq
```

### 4. Samba

Install and configure samba

```bash
dnf install policycoreutils-python-utils samba samba-common nano mc -y

systemctl enable smb nmb
systemctl start smb nmb

groupadd labusers
useradd -M -d /srv/samba/shared -s /usr/sbin/nologin labuser
usermod -g labusers labuser

smbpasswd -a labuser

#create directory structure

mkdir -p /data/smb_share
mkdir -p /data/smb_share/labIso

chmod -R 770 /data/smb_share/labIso
chown -R root:labusers /data/smb_share/labIso
chcon -t samba_share_t /data/smb_share/labIso

# without the two commands below - it is not possible to access samba shares via network
# Set the SELinux context for the shared directory to allow Samba to access it:

setsebool -P samba_export_all_ro on
semanage fcontext -a -t samba_share_t  "/data/smb_share/labIso(/.*)?"
restorecon -Rv /data/smb_share/labIso

# move the smb.conf
mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
nano /etc/samba/smb.conf
```

Paste following content into the smb.conf

```shell
[global]
workgroup = WORKGROUP
dos charset = cp850
unix charset = ISO-8859-1

log level = 2
dns proxy = no
# map to geust = Bad User - causes what windows can not access the share since some upgrade
# unless you perfrom some registry updates it just does not work
# seems it is better to reconfigure it on smb.conf
map to guest = Never
server string = Samba Server %v
netbios name = SAMBA-SERVER
security = USER
disable spoolss = yes
min protocol = SMB3
wins support  No
ntlm auth = true
idmap config * : backend = tbd

[labIso-smb]
comment = smb share xenserver
inherit acls = Yes
path =  /data/smb_share/labIso
valid users = @labusers root
browsable =yes
writable = yes
guest ok = no
read only = no

[labIso-nfs]
comment = smb share xenserver
inherit acls = Yes
path = /data/nfs_share/labIso
valid users = @labusers root
guest ok = no
writable = yes
browsable = yes
```

Restart smb, nmb services, configure firewall exception

```bash
# restart services
systemctl restart smb nmb
testparm
# configure firewall
firewall-cmd --permanent --zone=public --add-service=samba
firewall-cmd --reload
```

At this point, you should be able to reach the abovementioned paths over SMB

## Conclusions

Tested on:
* Rocky 9.4
* XCP-ng 8.2.1

Last update: 2024.06.24