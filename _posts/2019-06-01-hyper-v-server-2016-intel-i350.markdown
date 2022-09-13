---
layout: post
title: Hyper-V Server 2016 + Intel I350 = weird problems
date: '2019-06-01 12:05:00'
tags:
- networking
- windows
redirect_from:
- /hyper-v-server-2016-intel-i350
- /hyper-v-server-2016-intel-i350/
---

I have a two-node Hyper-V failover cluster where both nodes have a four-port gigabit PCI-e NIC installed for additional reduntant network ports. The first node has an older Intel PRO/1000, whereas the other one has the PRO/1000's new brother the I350. The PRO/1000 works fine, but the I350 has two issues I've had to (mostly) work around.

# Problem #1: VMQ

The I350 supports something called VMQ; Virtual Machine Queues. I won't be going deep into it but it essentially allows the NIC to directly transfer incoming Ethernet frames into a virtual machine's shared memory space with DMA. This is fine and all, except Windows trying to use it with the I350 breaks all VM network traffic in the card. There's no other visible symptom of it except the virtual machine just not getting any frames in or out. VMQ can be disabled per-VM in Hyper-V, or it can be disabled entirely for the card's individual interfaces.

<!--kg-card-begin: markdown-->

In Powershell, to disable a single VM network adapter's VMQ:

    Set-VMNetworkAdapter -Name <VM name> -VmqWeight 0

Or better, to disable a physical interface's VMQ:

    Set-NetAdapterAdvancedProperty <Interface name> -DisplayName "Virtual Machine Queues" -DisplayValue "Disabled"

<!--kg-card-end: markdown-->
# Problem #2: the card itself crapping out after a reboot

I don't actually know the reason behind this (maybe the card is really a counterfeit, who knows) but whenever the server soft-reboots, there's a high chance the card fails to come back up. This shows with the interfaces not showing up in Windows at all, and any NIC teams created out of them being invalid. This in turn means any VMs using the NIC team don't get any networking. The problem can be temporarily fixed by cold-booting the machine, but I haven't found any permanent solution.

It's not that big of a deal, except that it causes some very weird issues in the failover cluster. What I've ran into most is that live-migrating a VM into the host fails with "Not enough storage available to complete the operation", which you'll notice has absolutely fuck-all to do with the problem at hand. This has led me down a path of checking every part of any storage the machines might or might not have, which all have had plenty of space free, which has made the problem even more difficult to diagnose. The machine even appears to work fine from the outside, since its management network is using the onboard interfaces, not the card's. From what I've found, this error can arise from a lot of different things, which includes but isn't limited to the [target host not having enough memory available](https://www.virtualizationhowto.com/2017/03/windows-server-2016-hyper-v-cluster-not-enough-storage-space-available) or [printers bloating the registry](https://community.spiceworks.com/topic/161032-not-enough-storage-is-available-to-complete-this-operation-loaduserprofile). &nbsp;Now I can add "missing network adapter" to the possible causes.

