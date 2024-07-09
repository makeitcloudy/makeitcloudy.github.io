---
full-width: true
layout: post
title: "XCPng first VM from CLI"
permalink: "/xcpng-firstVM-from-CLI/"
subtitle: "First XCPng VM for XenOrchestra compiled from sources"
cover-img: /assets/img/cover/img-cover-xen.jpg
thumbnail-img: /assets/img/thumb/img-thumb-xpng8.png
share-img: /assets/img/cover/img-cover-xen.jpg
tags: [HomeLab ,Debian ,XCP-ng]
categories: [HomeLab ,Debian ,XCP-ng]
---
The hypervisor of choice is XCP-ng. This guide has not been invented by me, it is a written version of the material available on [YouTube - XCP-ng CLI creating a VM](https://www.youtube.com/watch?v=y4PJqvFCGZ0) by HomeTinyLab.


## Assumptions

+ fresh installation of XCP-ng - storage, network, ISO SR configured
+ scripts in /opt/scripts folder

## Prerequisites

Before starting the installation of the VM from the CLI, there are following prereq which should be met.

### Create local ISO storage repository

Run on XCPng.

```shell
### Create Local ISO storage
mkdir -p /var/opt/xen/ISO_Store
cd /var/opt/xen/ISO_Store
xe sr-create host-uuid=$host_UUID name-label="node4_hdd_LocalISO" type=iso device-config:location=/var/opt/xen/ISO_Store device-config:legacy_mode=true content-type=iso
export iso_SR_UUID=""
```

### Download the ISO

Download the iso which will be used for spinning up the VM.

```shell
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.5.0-amd64-netinst.iso
### Scan the Storage Repository
xe sr-list name-label="node4_hdd_LocalISO"
xe sr-scan uuid=$iso_SR_UUID
#xe sr-list type=iso
```

### Collect UUIDs

Run commands one by one, and collect the UUID's, unless you have a regex handy or skills to automate it.

```shell
### Get the XCP-ng UUID
xe host-list 
export host_UUID=""

### Get the Storage Repositories UUID
xe sr-list
export vm_SR_UUID=""

### Get the Network UUID
xe network-list
export network_UUID=""

### List the XCP-ng templates
xe template-list
xe template-list | grep Debian
```

## Virtual Machine Preparation

Setup the VM.

### Create VM skel

```shell
### Create VM
xe vm-install template="Debian Bookworm 12" new-name-label="debian-misc" sr-uuid=$vm_SR_UUID
export vm_UUID=""
xe vm-param-set uuid=$vm_UUID  name-description=" - node4 - [osName-version] - [function] - [IP Address]"

### Make use of the UUID identifier of the disk bound with the 'Disk 0 VDI'
xe vm-disk-list uuid=$vm_UUID
export vm_disk_UUID=""
### Resize the virtual disk of the VM
xe vdi-resize uuid=$vm_disk_UUID disk-size=32GiB
xe vdi-param-set uuid=$vm_disk_UUID name-label="debian-misc_osDisk" name-description="debian12-misc_osDisk"
```

### Configure VM

Run the code below in one go, the variables were collected above.

```shell
### List the iso with the debian string in it's name
xe cd-list | grep debian
export iso_fileName="debian-12.5.0-amd64-netinst.iso"
### Attach the installation media to the VM
xe vm-cd-add uuid=$vm_UUID cd-name=$iso_fileName device=1
### Make the installation media the boot device
xe vm-param-set HVM-boot-policy="BIOS order" uuid=$vm_UUID
### Link the network interface
xe vif-create vm-uuid=$vm_UUID network-uuid=$network_UUID mac='2A:47:41:D9:99:51' device=0
### Configure the memory of the VM
xe vm-memory-limits-set dynamic-max=2048MiB dynamic-min=2048MiB static-max=2048MiB static-min=512MiB uuid=$vm_UUID
### Start th VM
xe vm-start uuid=$vm_UUID
```

### Connect to the VM console from CLI

* Some distros starts the installation process quickly after powering up the VM, you may not have enough time to choose between the GUI of ASCII based installations
* When the VM is rebooted the vnc socket number seems to be arising by 1 (in example below, it's 35, after the reboot it's 36), you need to ammend the socat with the correct natural number afterwards
* make use Remmina or Remote Desktop Manager on Linux, on Windows VNC Viewer (localhost:9000)

```shell
# open second terminal tab on your device and set the SSH connection towards the XCP-ng
# you will need a second connection to preserve your variables within the existing SSH session
# socat will be running in the background - with this approach we are not using screen (multiplexing the terminal)

### socat should be installed to make it work
yum install socat

### Get the number of the VNC console we need to connect to
### The first number on the left is the vnc socket number
list_domains | grep $vm_UUID
socat TCP-LISTEN:9000 UNIX-CONNECT:/var/run/xen/vnc-35
### Leave the connection open
```

#### Linux 

Configure the SSH tunnel to the port where the hypervisor is listening on

```shell
# Get the IP address of XCP-ng MGMT interface and make use of it below
ssh -L 9000:[XCPng-MGMT_IP]:9000 root@[XCPng-MGMT_IP]
```

#### Windows

1. Putty -> Session - you have a configured Session connection towards your XCP-ng management interface.
2. Putty -> Connection -> Tunnels

Source port: 9000
Destination: [IP Address of the XCP-ng]:9000
Local|Auto

3. VNC Viewer -> localhost:9000
4. At this point you should see the installation screen of the VM.

### Connect to the VM console from CLI - after the VM reboot

```shell
### Run in the first SSH session towards XCP-ng (this session is where your variables are stored)
### After a reboot the VNC console session number increased by one
# get the number of the VNC connection
list_domains | grep $vm_UUID

### Run in the second SSH session towards XCP-ng (this session is for the SSH tunnel for the VNC to work)
socat TCP-LISTEN:9000 UNIX-CONNECT:/var/run/xen/vnc-37

### On your endpoint, run in terminal:
ssh -L 9000:[XCPng-MGMT_IP]:9000 root@[XCPng-MGMT_IP]

### On your endpoint launch Remmina and connect to localhost:9000

### Login to the VM as root
```

### Install XCP-ng tools

Run code on XCP-ng, in the SSH session where your variables are stored

```shell
### Run in the XCP-ng
### eject the installation media (debian-install.iso)
xe vm-cd-eject uuid=$vm_UUID

### insert guest_tools media
#xe cd-list | grep debian
xe vm-cd-insert cd-name=guest-tools.iso uuid=$vm_UUID
```

Run code on VM

```shell 
### run in VM
### mount cdrom and install guest-tools
mount /dev/cdrom /mnt
/mnt/Linux/install.sh

### Run in XCP-ng
### eject the installation media (guest-tools.iso)
xe vm-cd-eject uuid=$vm_UUID

### run in VM
reboot

### wait a minute

### Run in the first SSH session towards XCP-ng (this session is where your variables are stored)
list_domains | grep $vm_UUID
### Run in the second SSH session towards XCP-ng (this session is for the SSH tunnel for the VNC to work)
socat TCP-LISTEN:9000 UNIX-CONNECT:/var/run/xen/vnc-38

### On your endpoint launch Remmina and connect to localhost:9000
### Login as root
```

### Initial config of the VM

```shell
### Install openssh-server
apt install sudo nano openssh-server -y

### add user to the sudo group
#usermod -aG sudo sudoUser
nano /etc/sudoers
# modify the User privilege specification
sudoUser ALL=(ALL:ALL) ALL

### get the IP address of the VM
ip a
# modify DHCP leases
# ifdown enX0
# ifup enX0

### get the IP address of the VM on the XCP-ng
xe vm-list name-label="debian-misc" params=name-label,networks | grep -v "^$"
```

Open a regular SSH terminal, login as sudouser and continue the configuration.

### XCP-ng xe commands

```shell
xe vm-list params=name-label,power-state,networks,os-version,VCPUs-max,memory-static-max,disks,PV-drivers-up-to-date,PV-drivers-version uuid=$vm_UUID
```

## Summary

Tested on 2024.06.25
* hypervisor: XCP-ng 8.2.1
* VM: Debian 12.5
* endpoint: manjaro 24.0.1 Wynsdey

Last update: 2024.06.25
