---
layout: post
title: Installing Debian in LVM on an USB drive (and EFI booting it!)
date: '2018-11-27 18:06:00'
tags:
- homelab
- linux
redirect_from:
- /installing-debian-in-lvm-on-an-usb-drive-and-efi-booting-it
- /installing-debian-in-lvm-on-an-usb-drive-and-efi-booting-it/
---

This post is largely based on Will Haley's post over [here](https://willhaley.com/blog/install-debian-usb/). I've combined it with various Google searches to show how I built a minimal Debian image, ready to be written to an USB drive and booted off. We'll be using LVM as a middleman in the image to ease with future partition manipulation.

## Prerequisites

You'll need `debootstrap` to create the Debian installation, and its archive GPG keyring to validate the packages it'll download (you'll need `gpg` as well for the keyring). If you're on Debian or Ubuntu, you should have these already. I'm on Arch Linux, so my respective packages to install are called `debootstrap` and `debian-archive-keyring`. Your mileage may vary. For good measure, I grabbed `ubuntu-archive-keyring` as well.

## Image file

To ease the process, we'll create a disk image file to install Debian into, and later copy this to as many physical drives as we want.

Create the 0-filled image file. My USB drives are marketed at 16 gigs, in reality they show up at just over 14, so I'll be creating a 14 gigabyte image; adjust as needed. The minimum size is probably 2 gigabytes, depending on the software you install:

    $ fallocate -l 14G debian.img

Open it with fdisk:

    $ fdisk debian.img

fdisk automatically creates a BIOS (MBR) partition table in the image. We'll be using EFI, so create a new GPT partition table:

    g

Create the EFI boot partition, size it appropriately and set its type correctly (`EFI System`, identifier `1`). 200 to 250 mibibytes is the recommended minimum size for Debian's boot partition (once the image was complete, I found the bootloader to only take some hundred kilobytes):

    n 1 2048 +200M t 1

Create the LVM physical volume partition into the rest of the empty space (or make it a certain size, up to you) and change its type accordingly. Its type will be either in decimal or hexadecimal, see the list of types `L` if needed:

    n 2 <enter> <enter>
    t 2 31

We'll be creating more partitions as logical volumes inside this physical volume later so we don't need any more partitions here.

Write the changes:

    w

Create a [loop device](https://en.wikipedia.org/wiki/Loop_device) out of the image:

    # losetup --partscan --show --find debian.img

The command will output which loop device it mounted the image to, check that it and its corresponding partitions are found (for me, the device is `/dev/loop0` so the partitions are `/dev/loop0p1` and `/dev/loop0p2`):

    $ ls /dev/loop0*
    > /dev/loop0 /dev/loop0p1 /dev/loop0p2

The first partition is the EFI System, create a FAT32-filesystem in it. It's important you use FAT32, EFI won't work on anything else:

    # mkfs.fat -F 32 /dev/loop0p1

Create an LVM physical volume inside the other partition:

    # pvcreate /dev/loop0p2

Create a volume group out of the physical volume:

    # vgcreate debian-vg /dev/loop0p2

We'll be separating `/var` into its own five-gigabyte partition for future purposes, create a logical volume for it:

    # lvcreate -L 5G --name var debian-vg

If you want more logical volumes, create them appropriately now. Create the root logical volume into the rest of the empty space:

    # lvcreate -l +100%FREE --name root debian-vg

Create ext4-filesystems into the logical volumes (or whichever filesystem you favour). If the volume group doesn't show up as its own directory in `/dev`, the logical volumes should be available directly under `/dev/mapper`:

    # mkfs.ext4 /dev/debian-var/root
    # mkfs.ext4 /dev/debian-var/var

If you created any extra logical volumes, create filesystems in them as well now.

## Base image root

We'll mount the image under `/mnt/debian`, and into it we have to mount the special system directories from our host system.

Create the mount point for the root partition:

    # mkdir -p /mnt/debian

Mount the root logical volume:

    # mount /dev/debian-vg/root /mnt/debian

Create mount points for the rest of the logical volumes and the boot partition:

    # mkdir -p /mnt/debian/var
    # mkdir -p /mnt/debian/boot/efi

Mount them:

    # mount /dev/loop0p1 /mnt/debian/boot/efi
    # mount /dev/debian-vg/var /mnt/debian/var

Bootstrap the install. As we're building an EFI-booted install, 32-bit support is not straight-forward or even possible, so it's outside the scope of this post. To speed the package download a bit, pick a Debian mirror close to you from [the Debian mirror list](https://www.debian.org/mirror/list):

    # debootstrap --arch=amd64 --variant=minbase stretch /mnt/debian http://ftp.fi.debian.org/debian/

Mount special devices from your system into the image to create a complete system root:

    # mount -o bind /dev /mnt/debian/dev
    # mount -t proc /proc /mnt/debian/proc
    # mount -t sysfs /sys /mnt/debian/sys

`chroot` into it:

    # chroot /mnt/debian

## System setup

`chroot` inherits your environment variables from the host system into the chroot. Make sure your `PATH` environment variable contains `/bin`, `/usr/bin`, `/sbin` and `/usr/sbin`:

    # export PATH=$PATH:/bin:/usr/bin:/sbin:/usr/sbin

Install required packages for installing the rest of the system, and some additional handy utilities. If this step fails because of an unknown apt-key error, make sure your `PATH` is correct (see step above). Grub might try to install itself during this install, and will fail; it's okay, we'll install it properly later:

    # apt update
    # apt install linux-image-amd64 systemd-sysv grub2-common grub-efi lvm2 apt-utils
    # apt install iproute2 iputils-ping man vim dialog less

Add entries for the partitions `/etc/fstab`. It's important to identify the partitions with their labels, as their mountpoints and UUIDs may change across devices. We'll create these labels later, add their entries now:

    LABEL=DEBBOOT /boot/efi vfat defaults 0 0
    LABEL=DEBROOT / ext4 defaults 0 1
    LABEL=DEBVAR /var ext4 defaults 0 2

Edit `/etc/initramfs-tools/modules` to contain `lvm2`:

    # ...
    # Examples:
    #
    # raid1
    # sd_mod
    
    lvm2

Update the init RAM filesystem:

    # update-initramfs -u

These next few steps for grub will most likely complain about not being able to connect to `lvmetad`, you can ignore the warnings as the daemon doesn't exist in the chroot and is instead outside it in your host system.

Install grub to the mounted boot partition. The bootloader ID is largely irrelevant as the `--removable` flag tells the installer to not add its entry to the system's EFI variables (there are none in the chroot):

    # grub-install \
        --target=x86_46-efi \
        --efi-directory=/boot/efi \
        --bootloader-id=Debian \
        --force-file-id \
        --skip-fs-probe \
        --removable

Edit `/etc/default/grub` if you need to. I'll leave it as is now. Generate grub's config:

    # grub-mkconfig -o /boot/grub/grub.cfg

Grub by default might detect existing operating systems and allow you boot into them; it'll detect your host system and create entries for it. Delete them from `/boot/grub/grub.cfg`, or disable the feature file `/etc/grub.d/30_os-prober` and create a new config for grub.

Add additional Debian package repositories for security updates to `/etc/apt/sources.list`. Change the Debian mirror to something close to you, as previously done:

    deb http://ftp.fi.debian.org/debian stretch main
    deb-src http://ftp.fi.debian.org/debian stretch main
    
    deb http://security.debian.org/debian-security stretch/updates main
    deb-src http://security.debian.org/debian-security stretch/updates main
    
    deb http://ftp.fi.debian.org/debian/ stretch-updates main
    deb-src http://ftp.fi.debian.org/debian/ stretch-updates main

Do whatever you additionally need to at this point, such as setting root's password, setting the hostname and adding entries to the hosts-file:

    # passwd
    # cat "debian-usb" > /etc/hostname
    # cat "127.0.0.1 localhost" > /etc/hosts
    # cat "127.0.1.1 debian-usb" >> /etc/hosts

Finally exit the chroot:

    # exit

## Finalising the image

Unmount the image entirely:

    # umount -R /mnt/debian

As we're identifying the image partitions by their labels, we have to set them appropriately now:

    # fatlabel /dev/loop0p1 DEBBOOT
    # e2label /dev/debian-vg/root DEBROOT
    # e2label /dev/debian-vg/var DEBVAR

Deactivate the volume group:

    # vgchange -ay debian-vg

Delete the loop device:

    # losetup -d /dev/loop0

You can now write the image file into as many drives as you wish, each booting into a minimal Debian:

    # dd if=debian.img of=/dev/sdX bs=1M status=progress

## Moving on

### Booting

I found the boot process to complain about not being able to connect to `lvmetad` and failing `fsck` for the partitions, but moments later the daemon starts up and the system continues mounting the partitions just fine, and proceeds to give me a login prompt. `fsck` didn't seem to run though.

### Move partitions

The primary reason I used LVM and had `/var` in its own partition is when I've deployed the image to an USB drive and moved it permanently into a system, I want to move `/var` into proper mass storage as to not have the constant log writes in `/var/log` kill the USB drive. This is simply achieved by creating a new physical volume with `pvcreate`, extending the volume group with `vgextend`, and moving the logical volume with `pvmove`, where `/dev/sdb1` is the new physical volume and `/dev/sda2` is the existing one:

    # pvcreate /dev/sdb1
    # vgextend debian-vg /dev/sdb1
    # pvmove -n /dev/debian-vg/var /dev/sda2 /dev/sdb1

_Make sure the partition is inactive and unmounted, it's best to do this from a live boot if you're moving the root partition._

### Networking

We installed systemd to the image, which will handle network configuration for the OS. A simple configuration to get an address for an interface with DHCP would be as such:

Create a file called `<inet>.network` in `/etc/systemd/system/network/`. Replace `<inet>` with your primary interface's name (see `# ip link`). Fill the file with the following, again using the proper interface name:

    [Match]
    Name=<inet>
    [Network]
    DHCP=ipv4

Restart the networking service and enable it to start on boot:

    # systemctl restart systemd-networkd
    # systemctl enable systemd-networkd

Bring the network link up:

    # ip link set <inet> up

`systemd-networkd` will detect the link being up and assign it an address with DHCP.

### Time and date

The system's timezone can be configured with `dpkg-reconfigure`:

    # dpkg-reconfigure tzdata

You may want to optionally install `dialog` for a nicer terminal UI:

    # apt install dialog

### Keyboard and locale

Locales and keyboard are easily configured by first installing their respective packages for debconf;

    # apt install locales keyboard-configuration

They can be later adjusted with:

    # dpkg-reconfigure locales
    # dpkg-reconfigure keyboard-configuration

## Sources in no particular order

- [https://willhaley.com/blog/install-debian-usb/](https://willhaley.com/blog/install-debian-usb/)
- [https://help.ubuntu.com/community/DiskSpace](https://help.ubuntu.com/community/DiskSpace)
- `man mkfs.fat`
- `man debootstrap`
- [https://www.debian.org/mirror/list](https://www.debian.org/mirror/list)
- [https://wiki.archlinux.org/index.php/GRUB](https://wiki.archlinux.org/index.php/GRUB)
- [https://wiki.debian.org/GrubEFIReinstall](https://wiki.debian.org/GrubEFIReinstall)
- `man grub-install`
- [https://en.wikipedia.org/wiki/Loop\_device](https://en.wikipedia.org/wiki/Loop_device)
- [https://wiki.debian.org/LVM](https://wiki.debian.org/LVM)
- `man pvcreate`
- `man vgcreate`
- `man lvcreate`
- `man vgchange`
<!--kg-card-end: markdown-->