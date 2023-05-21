---
title: "Install PXE server on Raspberry Pi 4"
datePublished: Fri Oct 15 2021 12:08:57 GMT+0000 (Coordinated Universal Time)
cuid: ckusbv8br1smdebs103nv2knb
slug: install-pxe-server-on-raspberry-pi-4
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/pYpnKA52a-A/upload/f011e87686897e3f1c5cc9078cdf99d0.jpeg
tags: network, linux, devops, raspberry-pi

---

# About the plan

In this short tutorial, I will show how to boot live-cd-type systems over the network. We are going to use a Raspberry PI board as a server. For the operating system, I chose Raspberry Pi OS (early known as Raspbian) which is technically Debian 10 (Buster).

To set up a PXE Server we will need the following dependencies:

* TFTP
    
* dnsmasq
    
* NFS
    

Network boot is possible due to [TFTP](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol).

Usually, the TFTP server has the same IP address as the DHCP server, but we will use *dnsmasq* to configure some kind of relay for our goal. It is called proxy DHCP to be more precise. This way our setup will work in any local network even with grandma's router.

We will set up an NFS (Network File System) server, which will allow computers to access files on the PXE server over the network.

*Note: This tutorial assumes you are the* ***root*** *user, if not, please add* `sudo` for all the commands.

# Install and configure

The following command will install the required packages on Raspberry OS:

```bash
apt install -y dnsmasq pxelinux syslinux-common nfs-kernel-server
```

Once downloading is complete, stop the *dnsmasq* service

```bash
systemctl stop dnsmasq
```

## PXELINUX

Create a directory where all transferable files will reside

```bash
mkdir /srv/tftpboot
```

Take important boot modules (pxelinux.0, ldlinux.c32, libutil.c32, menu.c32 and vesamenu.c32) and place them into **/srv/tftpboot**

```bash
cp /usr/lib/PXELINUX/pxelinux.0 \
    /usr/lib/syslinux/modules/bios/{ldlinux.c32,libcom32.c32,libutil.c32,menu.c32,poweroff.c32,reboot.c32,vesamenu.c32} \
    /srv/tftpboot/
```

Create the directory for the PXE configuration file.

**Important**: this directory must be called `pxelinux.cfg`

```bash
mkdir -p /srv/tftpboot/pxelinux.cfg
```

Prepare boot menu design

```bash
nano /srv/tftpboot/pxelinux.cfg/default
```

with this content

```plaintext
MENU TITLE Network Boot Menu
UI vesamenu.c32
MENU INCLUDE pxelinux.cfg/pxe.conf

LABEL next
	MENU LABEL Load local bootloader
	MENU DEFAULT
	localboot

# Separator
MENU SEPARATOR

LABEL reboot
	MENU LABEL Reboot
	COM32 pxelinux.cfg/arch/reboot.c32

LABEL poweroff
	MENU LABEL Power Off
	COM32 pxelinux.cfg/arch/poweroff.c32
```

And styles separately

```bash
nano /srv/tftpboot/pxelinux.cfg/pxe.conf
```

```plaintext
MENU BACKGROUND pxelinux.cfg/logo.png
MENU RESOLUTION 1024 768
NOESCAPE 1
ALLOWOPTIONS 0
PROMPT 0
menu width 32
menu rows 5
MENU MARGIN 0
MENU VSHIFT 10
MENU HSHIFT 46
menu color title                1;37;40    #ffffffff #00000000 std
menu color border               36;40      #c00090f0 #00000000 std
menu color sel                  30;47      #00000000 #ffffffff all
menu color unsel                37;40      #ffffffff #00000000 std
```

And the background **/srv/tftpboot/pxelinux.cfg/logo.png**

![logo.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627461597527/6EueRlS2J.png align="left")

We will add real entries in the Test section.

### UEFI version

The setup of PXE boot for UEFI computers is slightly different from the setup supporting the BIOS computers. I will not cover it here, because the topic is very big for one article. In some rare cases, you will only need to find grubx64.efi file and replace `pxelinux.0` with `grubx64.efi`. Take it from the system that matches your client computers, for example, take it from the official [Ubuntu 20.04 (Focal) distribution](http://archive.ubuntu.com/ubuntu/dists/focal/main/uefi/grub2-amd64/current/). Then put `grubx64.efi` into `/srv/tftpboot/`. But usually, it's not that simple.

## dnsmasq

Update the content of **/etc/dnsmasq.conf**. By default this file full of comments above each option. You can read through and tweak settings to your liking, but I already know what I put into:

```bash
mv /etc/dnsmasq.conf /etc/dnsmasq.conf.example
nano /etc/dnsmasq.conf
```

```plaintext
#Disable DNS Server
port=0

#Enable DHCP logging
log-dhcp

#Respond to PXE request for the specified network;
#Run as DHCP proxy
dhcp-range=192.168.1.0,proxy,255.255.255.0

dhcp-boot=pxelinux.0

#Flag forces "simple and safe" behavior
dhcp-no-override

#Provide network boot option called "Network Boot"
pxe-service=x86PC,"Network Boot", pxelinux

#Turn tftp ON
enable-tftp

#Set tftp root
tftp-root=/srv/tftpboot
```

Add the following line to the **/etc/default/dnsmasq** file

```plaintext
DNSMASQ_EXCEPT=lo
```

## NFS

In order to give access to specific files to NFS clients add the following line to the **/etc/exports** file

```plaintext
/srv/tftpboot 192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)
```

And make this change live:

```bash
exportfs -a
```

# Test

For tests, we are going to run Kali Linux and custom Buildroot build (long story short, we'll assume that you already have **output/images/rootfs.tar.gz**)

## Kali

Download an ISO image from the [official website](https://www.kali.org/get-kali/#kali-live) and transfer it to Raspberry Pi (when I was writing this tutorial the last version was **2021.2** and the ISO name correspondingly **kali-linux-2021.2-live-amd64.iso**)

```bash
scp kali-linux-2021.2-live-amd64.iso pi@192.168.1.11:~
```

Then extract it to **/srv/tftpboot/kali**

```bash
mkdir -p /srv/tftpboot/kali
mount ~/kali-linux-2021.2-live-amd64.iso /mnt
cp -r /mnt/* /srv/tftpboot/kali
```

Add an entry to the boot menu

```bash
nano /srv/tftpboot/pxelinux.cfg/default
```

```plaintext
LABEL kali
    MENU vesamenu.c32
    MENU LABEL Kali Linux Live
    KERNEL /kali/live/vmlinuz
    APPEND root=/dev/nfs nfsroot=192.168.1.11:/srv/tftpboot/kali,tcp,v3 ip=dhcp vers=3 rootfstype=ext4 rw --
```

## Buildroot

Do not forget to enable in the kernel configuration

* CONFIG\_IP\_PNP\_DHCP=y
    
* NFS filesystem support (CONFIG\_NFS\_FS).
    
* The root file system on NFS (CONFIG\_ROOT\_NFS).
    

Transfer rootfs archive to Raspberry Pi

```bash
scp rootfs.tar.gz pi@192.168.1.11:~
```

Then extract it to **/srv/tftpboot/buildroot**

```bash
mkdir -p /srv/tftpboot/buildroot
tar xvpf ~/rootfs.tar.gz -C /srv/tftpboot/buildroot
```

Add to the boot menu an entry

```bash
nano /srv/tftpboot/pxelinux.cfg/default
```

```plaintext
LABEL buildroot
    MENU vesamenu.c32
    MENU LABEL Custom Buildroot build
    KERNEL /buildroot/boot/bzImage
    APPEND root=/dev/nfs nfsroot=192.168.1.11:/srv/tftpboot/buildroot,tcp,v3 ip=dhcp vers=3 rootfstype=ext4 rw --
```

# Credits and comparison

There are some great (?) tutorials out there describing the same topic, and of course, this one was compiled with their help.

Why this tutorial is better than

* [TecMint](https://www.tecmint.com/install-pxe-network-boot-server-in-centos-7/) - very descriptive, but here we use Debian's package manager
    
* [Debian's wiki](https://wiki.debian.org/PXEBootInstall) - focuses more on DHCP configuration, but we also add an NFS server and make a very nice boot menu
    
* [Red Hat](https://www.redhat.com/sysadmin/pxe-boot-uefi) - probably works great on Red Hat Linux, but I like free and open source solutions. So in this tutorial, we configure completely different packages
    

# Conclusion

Network boot is a long story. I've started writing a gitbook about it, so for more information on the topic you can [give it a read](https://neupokoev-n.gitbook.io/pxe-boot/).

Tags: #UEFI, #PXE, #network boot, #Raspberry Pi