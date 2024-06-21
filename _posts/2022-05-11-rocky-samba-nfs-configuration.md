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

## Assumptions

XCP-ng:

* /var/opt/xen/ISO_Store - contains the Rocky9 ISOroot
* /opt/scripts - contains the vm_create_bios.sh script
* /opt/scripts - contains the vm_add_disk.sh script

```bash
# run code in the XCP-ng terminal
/opt/scripts/vm_create_bios.sh --VmName 'rocky9FS' --VCpu 4 --CoresPerSocket 2 --MemoryGB 4 --DiskGB 32 --ActivationExpiration 0 --TemplateName 'Rocky Linux 9' --IsoName 'Rocky-9.2-x86_64-minimal.iso' --IsoSRName 'hdd_LocalISO' --NetworkName 'eth1 - VLAN1342 untagged - up' --Mac '2A:47:41:D9:00:50' --StorageName 'node4_ssd_sdd' --VmDescription 'node4_rocky9_nfs_smb'
```

## Installation

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

### Initial Configuration

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

### Add drive and volume

In XCP-ng terminal, run code

```bash
# add extra disk - dedicated for NFS and SMB storage
/opt/scripts/vm_add_disk.sh --vmName "rocky9FS" --storageName "node4_hdd_sdc_lsi" --diskName "00_rocky9_filer_data_disk" --deviceId 4 --diskGB 160  --description "00_rocky9_filer_nfs_smb"
```

In Rocky9 VM, run code

```bash
dnf install policycoreutils-python-utils nano mc -y

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

### Samba

Install and configure samba

```bash
dnf install samba samba-common -y
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

semanage fcontext -a -t samba_share_t  "/data/smb_share/labIso(/.*)?"
restorecon -Rv /data/smb_share/labIso
```

### smb.conf

Move the smb.conf

```bash
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

### Restart Services

Restart smb, nmb services

```bash
systemctl restart smb nmb
testparm
```

### Configure Firewall

Configure Firewall exception

```bash
firewall-cmd --permanent --zone=public --add-service=samba
firewall-cmd --reload
```

At this point, you should be able to reach the abovementioned paths over SMB


## Troubleshoot

```bash
   40  sudo dnf install policycoreutils-python-utils
   41  sudo mkdir -p /data/smb_share/labIso
   42  sudo chown -R root:labuser /data/smb_share/labIso
   43  sudo chmod -R 0750 /data/smb_share/labIso
   44  sudo semanage fcontext -a -t samba_share_t "/data/smb_share/labIso(/.*)?"
   45  sudo restorecon -Rv /data/smb_share/labIso
   46  sudo setsebool -P samba_export_all_ro on
   47  nano /etc/samba/smb.conf
   48  smbpasswd -a labuser
   49  systemctl restart smb nmb
   50  sudo setsebool -P samba_export_all_ro on
   51  sudo semanage fcontext -a -t samba_share_t "/data/smb_share/labIso(/.*)?"
   52  sudo restorecon -Rv /data/smb_share/labIso
   53  systemctl restart smb
   54  systemctl restart nmb
   55  nano /etc/samba/smb.conf
   56  testparm
   57  sudo systemctl restart smb
   58  sudo systemctl restart nmb
   59  sudo chown -R root:labusers /data/smb_share/labIso
   60  sudo chmod -R 0750 /data/smb_share/labIso
   61  history

```

## Conclusions

Tested on:
* Rocky 9.4
* XCP-ng 8.2.1

Last update: 2024.06.21