---
layout: post
title: "Rocky9 Samba and NFS configuration"
permalink: "/rocky-samba-nfs-configuration/"
subtitle: "Rocky9 Samba na NFS setup - ISO repository for XCP-ng"
cover-img: /assets/img/cover/img-coer-linux-rocky.jpg
thumbnail-img: /assets/img/thumb/img-thumb-rocky.png
share-img: /assets/img/cover/img-cover-linux-rocky.jpg
tags: [HomeLab , Rocky, XCP-ng]
categories: [HomeLab , Rocky, XCP-ng]
---

Here are the steps how to set up Rocky9 as Samba and NFS share for XCP-ng and it's VMs.

## Links

* [How To Create a New Sudo-enabled User on Rocky Linux 8 [Quickstart]](https://www.digitalocean.com/community/tutorials/how-to-create-a-new-sudo-enabled-user-on-rocky-linux-8-quickstart)
* [How to Install Samba Server on Rocky Linux 9 / AlmaLinux 9](https://www.linuxbuzz.com/install-samba-server-on-rockylinux-almalinux/)

## Assumptions

XCP-ng:

* /var/opt/xen/ISO_Store - contains the Rocky9 ISO
* /opt/scripts - contains the vm_create_bios.sh script
* /opt/scripts - contains the vm_add_disk.sh script

```bash
# run code in the XCP-ng terminal
/opt/scripts/vm_create_bios.sh --VmName '00_roky9_filer' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 0 --TemplateName 'Rocky Linux 9' --IsoName 'Rocky-9.2-x86_64-minimal.iso' --IsoSRName 'hdd_LocalISO' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:00:00' --StorageName 'node4_hdd_sdc_lsi' --VmDescription 'node4_rocky9_filer'
```

## Installation

1. Install Rocky Linux
2. Set Root password
3. Installation Destination -> pick the base disk
3.1 Begin Installation
3.2 Reboot system
4. login
5. eject the installation media
6. mount *guest-tools.iso* to rocky9 vm

### Initial Configuration

Run the initial configuration commands via XenOrchestra virtual terminal. At this point by default you won't be able to login via ssh to the VM.

```bash
# rename the hostname
hostnamectl
hostname 00-rocky9
# update
yum update
# add extra user with sudo permissions
adduser labuser
passwd labuser
usermod -aG wheel labuser
su - labuser

# install XCP-ng tools
blkid
mkdir /media/cdrom
mount /dev/sr0 /media/cdrom
cd /media/cdrom/Linux
# install xcp-ng tools
bash install.sh -d rhel -m 9
cd ~
umount /dev/sr0 /media/cdrom
```


### Add drive and volume

```bash
# run code in the XCP-ng terminal
# add extra disk - dedicated for NFS and SMB storage
/opt/scripts/vm_add_disk.sh --vmName "00_roky9_filer" --storageName "node4_hdd_sdc_lsi" --diskName "00_rocky9_filer_data_disk" --deviceId 4 --diskGB 160  --description "00_rocky9_filer_nfs_smb"
```

```bash
# run code in the rocky VM
mkdir /data
ls -lah /dev/xvd*
fdisk -l
fdisk /dev/xvde
# here you get into the partitioning of the volume you added to the VM
# Command: n
# Command: p
# Command: 1
# Command: w

# create a filesystem
mkfs.ext4 -f /dev/xvde

#mount the filesystem
mount /dev/xvde /data

vi /etc/fstab
# INSERT
# /ect/fstab contains following entry
/dev/xvde       /data   ext4    defaults     0   0
# ESC
# :wq
```

### Samba

Install and configure samba

```bash
dnf install samba samba-common -y
systemctl enable smb nmb
systemctl start smb nmb

groupadd labusers
useradd -g labusers labuser

useradd -M -d /srv/samba/shared -s /usr/sbin/nologin labuser
smbpasswd -a labuser

nano /etc/samba/smb.conf

#create directory structure

mkdir -p /data/smb_share
mkdir -p /data/smb_share/labIso

mkdir -p /data/smb_share/labIso
chmod -R 770 /data/smb_share/labIso
chown -R root:labusers /data/smb_share/labIso
chcon -t samba_share_t /data/smb_share/labIso
```

### smb.conf

```bash
nano /etc/samba/smb.conf
```

```shell
[global]
workgroup = WORKGROUP
dos charset = cp850
unix charset = ISO-8859-1

log level = 2
dns proxy = no
map to guest = Never
server string = Samba Server %v
netbios name = SAMBA-SERVER
security = USER
disable spoolss = yes
min protocol = SMB2
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

### Restart Services

```bash
systemctl restart smb nmb
testparm
```

### Configure Firewall

```bash
firewall-cmd --permanent --zone=public --add-service=samba
firewall-cmd --reload
```

At this point, you should be able to reach the abovementioned pahts over SMB

## Conclusions

Last update: 2024.06.21