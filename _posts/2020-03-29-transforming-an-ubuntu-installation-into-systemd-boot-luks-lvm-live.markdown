---
layout: post
title: Transforming an Ubuntu installation into systemd-boot + XanMod + LUKS + LVM
featured: true
date: '2020-03-29 14:48:47'
tags:
- linux
redirect_from: /transforming-an-ubuntu-installation-into-systemd-boot-luks-lvm-live
---

My laptop's been running standard Ubuntu 19.04 for some time now. I had the idea of swapping its bootloader, GRUB2, for systemd-boot, using the XanMod kernel and swapping its bunch-of-GPT-partitions into LUKS + LVM.

Motivation? I was bored as fuck during this self-imposed quarantine.

Quick word of warning, this post is in no way written to be a complete guide on how to achieve what I described step-by-step, but more of a writeup on what I did from which you can learn and adapt.

## Starting off

The initial partitions look as such:

| Partition | Size | Filesystem | Mountpoint |
| --- | --- | --- | --- |
| sda1 | 250M | EFI System | /boot/efi |
| sda2 | 8G | swap | [SWAP] |
| sda3 | 560G | ext4 | /home |
| sda4 | 50G | ext4 | /var |
| sda5 | 380G | ext4 | / |

Pretty standard stuff. What I'm aiming to achieve here is to get something like:

| Partition | Size | Filesystem | Mountpoint |
| --- | --- | --- | --- |
| sda1 | 250M | EFI System | /boot/efi |
| sda2 | ~1TB | LUKS | |
| -\> LVM | | LVM | |
| -\> -\> LV root | 60G | ext4 | / |
| -\> -\> LV | 60G | ext4 | /var |
| -\> \> LV home | ~800G | ext4 | /home |
| -\> -\> LV swap | 16G | swap | [SWAP] |

A bit more complicated, sure, but where's fun in simple?

The two required packages for all this are `lvm2` and `cryptsetup`.

Let's start off with the easy bit.

## systemd-boot

I'll be closely following [this guide by Josh Stoik](https://blobfolio.com/2018/06/replace-grub2-with-systemd-boot-on-ubuntu-18-04/), adapted to my scenario.  
First up, change to the root user to ease things up a bit: `sudo -i`

Create a skeleton directory structure for the new bootloader.

    cd /boot/efi
    mkdir -p loader/entries
    mkdir ubuntu

The `loader` and `entries` directories will contain configuration files for each thing you wish to boot, and the `ubuntu` directory will contain the kernel and initramfs images.

Then the initial loader configuration file. This defines the default boot, lets the boot selection screen be displayed for a second before automatically selecting the default, and allow the user to change the boot parameters before booting.

> `loader/loader.conf`

    default ubuntu
    timeout 1
    editor 1

‌  
As Stoik explains in his post, Debian-based systems won't automatically get its generated kernel and initramfs images into this partition, which means you'll have to do it yourself. Likewise, systemd-boot won't generate entries for each kernel you might use so you'll have to create those yourself as well. Stoik provides a script for it which I've adapted for my use.

    #!/bin/bash
    #
    # This is a simple kernel hook to populate the systemd-boot entries
    # whenever kernels are added or removed.
    #
    # Original script by Josh Stoik, modified by Spans
    # 
    
    # The UUID of your encrypted partition.
    UUID="CHANGEME"
    
    # The LUKS volume slug you want to use, which will result in the
    # partition being mounted to /dev/mapper/CHANGEME.
    VOLUME="CHANGEME"
    
    # Any rootflags you wish to set.
    ROOTFLAGS="quiet splash"
    
    # Our kernels.
    KERNELS=()
    FIND="find /boot -maxdepth 1 -name 'vmlinuz-*' -type f -print0 | sort -rz"
    while IFS= read -r -u3 -d $'\0' LINE; do
    	KERNEL=$(basename "${LINE}")
    	KERNELS+=("${KERNEL:8}")
    done 3< <(eval "${FIND}")
    
    # There has to be at least one kernel.
    if [${#KERNELS[@]} -lt 1 ]; then
    	echo -e "\e[2msystemd-boot\e[0m \e[1;31mNo kernels found.\e[0m"
    	exit 1
    fi
    
    # Perform a nuclear clean to ensure everything is always in perfect
    # sync.
    rm /boot/efi/loader/entries/*.conf
    rm -rf /boot/efi/ubuntu
    mkdir /boot/efi/ubuntu
    
    # Copy the latest kernel files to a consistent place so we can keep
    # using the same loader configuration.
    LATEST="${KERNELS[@]:0:1}"
    echo -e "\e[2msystemd-boot\e[0m \e[1;32m${LATEST}\e[0m"
    for FILE in config initrd.img System.map vmlinuz; do
        cp "/boot/${FILE}-${LATEST}" "/boot/efi/ubuntu/${FILE}"
        cat << EOF > /boot/efi/loader/entries/ubuntu.conf
    title Ubuntu
    linux /ubuntu/vmlinuz
    initrd /ubuntu/initrd.img
    options cryptdevice=UUID=${UUID}:${VOLUME} root=/dev/mapper/crypt--vg-root rw ${ROOTFLAGS}
    EOF
    done
    
    # Copy any legacy kernels over too, but maintain their version-based
    # names to avoid collisions.
    if [${#KERNELS[@]} -gt 1 ]; then
    	LEGACY=("${KERNELS[@]:1}")
    	for VERSION in "${LEGACY[@]}"; do
    	    echo -e "\e[2msystemd-boot\e[0m \e[1;32m${VERSION}\e[0m"
    	    for FILE in config initrd.img System.map vmlinuz; do
    	        cp "/boot/${FILE}-${VERSION}" "/boot/efi/ubuntu/${FILE}-${VERSION}"
    	        cat << EOF > /boot/efi/loader/entries/ubuntu-${VERSION}.conf
    title Ubuntu ${VERSION}
    linux /ubuntu/vmlinuz-${VERSION}
    initrd /ubuntu/initrd.img-${VERSION}
    options cryptdevice=UUID=${UUID}:${VOLUME} root=/dev/mapper/crypt--vg-root rw ${ROOTFLAGS}
    EOF
    	    done
    	done
    fi
    
    exit 0

This script will be used as a hook when kernel images are installed or updated. Copy it into both `/etc/kernel/postinst.d/zz-update-systemd-boot` and `/etc/kernel/postrm.d/zz-update-systemd-boot` and make them executable (permissions `0755`).

Then install the new bootloader: `bootctl install --path=/boot/efi`

Stoik's guide goes into configuring the bootloader for Secure Boot, but I'll be skipping that for now. Test that the new bootloader works, and when it does you can safely purge GRUB: `apt purge grub*`

## XanMod kernel

This bit's simple. Add XanMod's apt source and signing key.

    echo 'deb http://deb.xanmod.org releases main' | sudo tee /etc/apt/sources.list.d/xanmod-kernel.list
    wget -qO - https://dl.xanmod.org/gpg.key | sudo apt-key add -

Then install it.

    apt update
    apt install linux-xanmod

The kernel `postinst`- and `postrm`-hooks set up earlier will make sure the bootloader has the kernel and initramfs images available and that the proper bootloader entries are in place.

## Pivoting to LUKS + LVM

This bit's not simple. Since I'm so adventurous I decided to do all this without resorting to booting into a live media. On a high-level my approach is the following:

1. shrink the existing root partition enough to fit a copy of all data on the system
2. set up the new partition and encryption scheme on the newly created empty space
3. copy everything over to the new scheme
4. make it bootable
5. once it works, delete the old partitions
6. grow the new partitions to fill the drive

### Get the old root unmounted

Shrinking an ext4 partition can be done without reboots if the partition is first unmounted. Just directly unmounting the root partition isn't going to work for obvious reasons, which is why it's commonplace to use a bootable media and shrink the partition from there. I mentioned I wasn't going to use any bootable media, so how exactly am I going to unmount my root partition while still booted from it?

Enter `pivot_root`, a thing about as magical as `chroot`. What it does is change the active root into some other location, just like `chroot`, but also mount the old root into a directory inside the new one. From there you can still mess with the old root while residing in the new one. I'll be following and adapting [this guide by Tom Hunt](https://unix.stackexchange.com/a/227318).

First things first, there's no way to do all the tricks here if there's an entire desktop environment running. Start off by booting into single-user mode. With the bootloader we set up earlier, this is easily done by pressing `e` during the boot selection and appending `single` to the boot parameters. This'll make systemd start the minimum required services and run an emergency rescue shell that lets you access everything you'll need.

What we'll be doing next is copying enough of the system into a memory-backed tmpfs, pivoting into it and unmounting the root filesystem on drive, after of which the drive's partitions are free to mess around with.

Create a location for the pivot root and mount a tmpfs into it. Size the tmpfs appropriately to fit the important bits of your root partition. Keep in mind it'll reside entirely in memory and use swap if needed.

    mkdir -p /tmp/tmproot
    cd /tmp
    mount -t tmpfs -o size=14G none tmproot

Copy the important bits over to it.

    mkdir /tmp/tmproot/{proc,sys,dev,run,usr,var,tmp,oldroot}
    cp -ax /{bin,etc,mnt,sbin,lib,lib64} tmproot/
    cp -ax /usr/{bin,sbin,lib,lib64} tmproot/usr/
    cp -ax /var/{account,empty,lib,local,lock,nis,opt,preserve,run,spool,tmp,yp} tmproot/var

I had an issue where the combined size of my root + `/var` was larger than I could fit into memory + swap (16G total), so I opted to leave `/var` out from the pivot. This turned out to not cause any issues - or in any case I didn't notice or run into any.

Pivot into the new root and mount the system filesystems into it.

    mount --make-rprivate /
    pivot_root /tmp/tmproot /tmp/tmproot/oldroot
    for i in dev proc sys run; do mount --move /oldroot/$i /$i; done

Now to get the old root partition unmounted. Everything running off it has to be stopped (or killed). See what's using it and what services are currently running.

    fuser -vm /oldroot systemctl | grep running

`systemctl stop` the services, or kill the executables. Before you do anything there's two catches though!

- systemd itself is still running off the old root. This is easily fixed by re-execing it with systemctl daemon-reexec
- The rescue session you're in is also running off the old root. Stopping or killing it will end the session and you'll have to force-boot your machine to get back, nullifying all your progress. Instead of stopping or killing it, restart the session: `systemctl restart rescue`

Once the old root isn't being used by anything anymore, unmount it: `umount /oldroot`. If like me you have `/var` and `/home` mounted on it, unmount them as well before finally unmounting the old root.  
‌

### Shrink it, crypt it, LVM it

Now that the old root's unmounted and we're running off memory, the old root is easily shrunk.

    e2fsck -f /dev/sda5
    resize2fs -L 30G /dev/sda5

This frees up a neat bit of space to work in. Using a decent partition editor (I'm using `fdisk`), delete the root partition and recreate it as large (or small?) as the filesystem was shrunk to. Then create a new partition in the empty space.

Encrypt and open this newly created partition (_pick a strong passphrase!_).

    cryptsetup luksFormat --type=luks2 /dev/sda6
    cryptsetup open /dev/sda6 lvmcrypt

Set it up as an LVM physical volume, stick a volume group into it and get some logical volumes and filesystems going. Standard LVM stuff here, I won't be going to much detail.

    pvcreate /dev/mapper/lvmcrypt
    vgcreate crypt-vg /dev/mapper/lvmcrypt

    lvcreate -L 60G -n root crypt-vg
    lvcreate -L 60G -n var crypt-vg
    lvcreate -L 2G -n swap crypt-vg
    lvcreate -l 100%FREE -n home crypt-vg

    mkfs.ext4 /dev/mapper/crypt--vg-root
    mkfs.ext4 /dev/mapper/crypt--vg-var
    mkfs.ext4 /dev/mapper/crypt--vg-home
    mkswap /dev/mapper/crypt--vg-swap

There's the new partitions, now to get data from the old ones into them.

### The great migration

Get the old root back into business, and any other partitions mounted inside it.

    mount /dev/sda5 /oldroot
    mount /dev/sda4 /oldroot/var
    mount /dev/sda3 /oldroot/home

Get the new partitions into play as well.

    mkdir /newroot
    mount /dev/mapper/crypt--vg-root /newroot
    mkdir /newroot/{var,home}
    mount /dev/mapper/crypt--vg-home /newroot/home
    mount /dev/mapper/crypt--vg-var /newroot/var

Simple copies from there to here will do, much like how previously was done. This'll take a while!

    cp -ax /oldroot/{bin,boot,sbin,etc,lib,lib64,libx32,opt,root,snap,srv,usr,var,home} /newroot/

Nothing better than stacking magic on top of magic; `chroot` into this new root.

    for i in dev sys proc run; do mount --bind /$i /newroot/$i; done
    chroot /newroot

We've now gone from the pre-existing root partition into an in-memory root into the new root. Pretty wild!

### Boot it

Now that there's LUKS in between the drive and its data, there's a bit of configuration for it in order to make it work at boot.

Find out the UUID of the encrypted partition: `blkid /dev/sda6`. Stick that and the crypt device's name into `/etc/crypttab`.

> `/etc/crypttab`

    # <target name>	<source device> <key file>	<options>
    lvmcrypt UUID=CHANGEME none luks,discard

Jam the UUID into the script from earlier as well if you haven't already!

Generate new initramfs and set up the bootloader.

    update-initramfs -u -k all
    /etc/kernel/postinst.d/zz-update-systemd-boot

The new root is now complete, including the bootloader and all. Cross your fingers and reboot the system.

## Get some space

If all went well (somehow I doubt that) the system will boot into a prompt asking for the decryption key for the encrypted partition. Once that's open, LVM steps in and activates the logical volumes inside the now-unlocked partition. From there on systemd does its thing and starts everything required.

There's still the matter of reclaiming the space left behind by the old partitions.

### Space to move into

Get rid of the old partitions and create a single new one to fill the space they left behind. Encrypt and open it. Create an LVM PV out of it. Extend the existing LVM VG with it.

    cryptsetup luksFormat --type=luks2 /dev/sda2
    cryptsetup open /dev/sda2 newcrypt
    lvcreate /dev/mapper/newcrypt
    vgextend crypt-vg /dev/mapper/newcrypt

Move the entire previous PV into the new one. This too will take a while!

    pvmove /dev/mapper/crypt /dev/mapper/newcrypt

Remove the old PV from the VG. Close and delete it. Expand the new one to fill its space.

    vgreduce crypt-vg /dev/mapper/lvmcrypt
    cryptsetup close /dev/mapper/lvmcrypt
    fdisk /dev/sda
    ...

Extend the partition, the crypt device and the PV to this new space.

    fdisk /dev/sda
    ...
    cryptsetup resize /dev/mapper/newcrypt
    pvextend /dev/mapper/newcrypt

### Booting's broke, fix it

This new partition has a different UUID to the previous one (duh), change it in both `/etc/crypttab` and the systemd-boot hook scripts.

    blkid /dev/sda2

> `/etc/crypttab`

    # <target name>	<source device> <key file>	<options>
    lvmcrypt UUID=NEWUUID none luks,discard

> `/etc/kernel/postinst.d/zz-update-systemd-boot` `/etc/kernel/postrm.d/zz-update-systemd-boot`

    ...
    # The UUID of your encrypted partition.
    UUID="NEWUUID"
    ...

Recreate initramfs and set up the bootloader

    update-initramfs -u -k all
    /etc/kernel/postinst.d/zz-update-systemd-boot

That's it! Enjoy your new encrypted, LVM'd, systemd-boot'd and XanMod'd Ubuntu.

<!--kg-card-end: markdown-->