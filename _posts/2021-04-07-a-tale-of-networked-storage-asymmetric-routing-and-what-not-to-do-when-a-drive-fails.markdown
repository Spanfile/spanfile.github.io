---
layout: post
title: A tale of networked storage, asymmetric routing and what not to do when a drive
  fails
date: '2021-04-07 08:13:52'
tags:
- homelab
redirect_from:
- /a-tale-of-networked-storage-asymmetric-routing-and-what-not-to-do-when-a-drive-fails
- /a-tale-of-networked-storage-asymmetric-routing-and-what-not-to-do-when-a-drive-fails/
---

My lab network has a centralised storage server, a Dell R510 with a mismash of drives, running Debian 10 and ZFS. So far it has shared this storage over NFS, but that has turned out to cause issues which is why I opted to change the sharing medium to my lovechild iSCSI.

## What caused the change

In addition to the three ESXi hypervisors using that storage, there are a couple of virtual machines also mounting a share in that server over NFS for their own storage needs. For a couple weeks now, none of the virtual machines have had functioning NFS. I have a hunch it's an internal bug, caused by a version mismatch between the server and the clients, but I haven't figured out anything concrete for it. In order to debug it further, I'd have to mess with the NFS server running in the storage server, which isn't very ideal given every single VM I have runs off that NFS server. This is why I opted to change the VMs to run off an iSCSI target, which leaves the NFS isolated and ready to be tampered with.

## Migrating to iSCSI

After I got each hypervisor to mount the newly created iSCSI target on a ZFS zvol, it was time to start migrating all VMs to it. It has to happen via the hypervisors, since they can do it live while the VMs are running. I could transfer the VM files all locally in the storage server, but it'd require disassociating them from ESXi and later registering them again, which is a hassle. There's a ten gigabit network between the storage server and the hypervisors, so the transfer will go as fast as the drives allow..

right?

## The asymmetric footgun

I began migrating some offline VMs first to see how well they move from one storage medium to.. another? It's really a copy from the drives back to themselves, just that the protocol changes. Anyways. It was going a bit slow, but surely that's to be expected. I looked at the ten gigabit switch statistics to see about how well the transfer is going based on the network activity, and I noticed something peculiar.

All the traffic was being put through the switch's gigabit uplink to my gigabit switch, which is used largely for management network access for OOB devices and such. It immediately clicked what was wrong; the storage server has a gigabit interface in the management network, and a ten gigabit interface also in the same management network. I know, I know, storage and management don't mix, it's about to change and I've learned my lesson. While the hypervisors were mounting the NFS share over the ten gigabit network, the storage server was responding to it over the gigabit network, which essentially bottlenecked the entire storage access to a single gigabit link. Ten gigabit one way, a single gigabit the other. This also meant that the iSCSI targets had the same gun stuck to their feet. This is a classic issue solvable by policy routing, but I've never learned policy routing nor have I actually bothered learning it. This was not the time to learn it though.

It was an easy fix. I created a new VLAN for the storage network, put it strictly into the ten gigabit network and remounted the iSCSI target over it. The NFS was still in the old mess, since changing it would require shutting every VM down, remounting the share and so on and so on, not something I would've bothered to do. So I just sucked it up and started migrating VMs over the gigabit bottleneck. No worries, I had time, nothing to worry about.

## Why do drives always fail at the critical moments?

Except I had to worry about drive failure. While migrating the VMs, it suddenly paused for about five seconds, then continued on like nothing had happened. I looked at vCenter, it had no complaints. I looked at the storage server, and one drive's LED had stopped blinking. `zpool status` confirmed my suspicion; a drive had failed enough writes so ZFS took it offline as unavailable, marked the pool as degraded and continued on. This was the momentary pause in the transfer as ZFS recovered from the fault. At that point if another drive failed, catastrophic data loss would've occurred, so I had to figure out something to restore the pool.

Looking at the kernel log messages, it seemed as if the drive had lost power from the storage server's backplane, and afterwards returned functional (under a different drive device of course). Since the drive was offline from ZFS's point of view, I could be sure ZFS wouldn't touch it before I tell it to. I ran a short SMART test on it, which didn't reveal any faults. The drive's SMART statistics were fine as well, no reallocated sectors or anything such. Of course, it's a consumer drive and with consumer drives, SMART is at least a week late in its information and is more useful in confirming that an obviously dead drive is dead.

## What not to do when a drive fails

So I decided to do something silly, and replace the drive in the pool with itself. This is a monkey-see-monkey-do kinda situation, it should go without saying that replacing a possibly faulty drive with another faulty drive often doesn't end well. Except it did in this case, but give it time and it probably fails again. I emptied the drive's partition table and all &nbsp;ZFS signatures, since ZFS refuses to use a drive it sees is already a part of another ZFS pool. After telling ZFS to replace the faulty drive with the "new" one, I began the arduous journey of waiting for the resilver to finish. This is also where computers got to shine with their ETAs again, since this process' ETA began at an hour, dropped down to 20 minutes and then began rising steadily up until the process was done.

And yeah, it did finish succesfully without more errors, all the while the VM migration was taking place. I'd already started migrating live VMs, and at best there were maybe 15 simultaneous migrations. The network handled it well, ZFS handled it well, all's well that ends well.

I've already ordered replacement drives, since there are two identical drives as the faulty one. I also have plans to reconfigure the entire thing, possibly migrate to vSAN if I can get my hands on some SSDs. Yep, it'll be one more migration process, although that time actually over the ten gigabit network.

