---
layout: post
title: Why I'm not recommending VyOS anymore and how I built my own
date: '2019-07-15 07:48:00'
tags:
- networking
- linux
redirect_from: /why-im-not-recommending-vyos-anymore
---

I used to like [VyOS](https://vyos.io/'') quite a lot as a general-purpose lightweight Linux-based easily virtualised router appliance. It didn't really have anything to complain about, other than the fact that the latest build for its version 1.1.8 was released late-2017.

Starting October 2018 it's getting too old. Components start acting in obscure ways when interacting with their newer counterparts (from experience its use of a very old strongSwan caused plentiful issues). Luckily the development for the new version 1.2.0 has been underway and its first release candidate is out. Great! Packages are updated, new features are added. Too bad that it'll take them [11 release candidates](https://blog.vyos.io/vyos-1.2.0-rc11-is-available-for-download) to get it stable enough with the new unstable build of FRR and whatnot. Somewhere in that time they also decided to start making money with VyOS and moved the stable builds behind a paywall. You can get the stable branch, Crux, free if you're a non-profit of sorts, or are willing to build it from source yourself. The free builds, as they call, are "rolling" releases.

# How (not) to do rolling releases

In the software development world rolling releases refer to the model of releasing updates as soon as they're available, functioning and stable. VyOS' rolling releases are nightlies that sure are available, but functioning is a roll of a die and stability is obviously not a thing or otherwise they'd be behind the aforementioned paywall. They're not even nightlies of the very latest Crux version 1.2.1, but instead nightlies of the already outdated 1.2.0. This is the main reason why I'm not recommending VyOS anymore: the builds you get outside the paywall are out of date despite being nightlies, and very, very broken. Unfortunately the only somewhat reasonable, yet still absolutely nonsensical solution to it I've heard has been "find one nightly that works for you and use it everywhere, never updating in fear of it not working anymore". I'm sure you see why that is not a good solution.

# How I built my own replacement

In its essence VyOS is Debian 8 (with an updated kernel), FRR for dynamic routing, iptables for firewalling, a unifying configuration utility and a handful of network utilities usually seen in routers. Installing those (except the configuration utility) into a standard Debian install isn't difficult at all, which is why that's exactly what I did.

## Interface configuration
<!--kg-card-begin: markdown-->

At a glance the only features my network used with its handful of VyOS installs were IBGP, static routing, firewalling and VRRP. Starting with a Debian 9 install (at the time of writing 10 was still just moments away from final release, in fact it was released in the middle of writing), I began by getting a bit more modern network configuration. Debian 9 is sticking to `ifupdown` and likely upgrading it to `ifupdown2` in Buster. I decided to slightly shrink the installation size by ditching `ifupdown` and using SystemD's networking configuration, which is also more flexible than `ifupdown`. Its configuration is located in `/etc/systemd/networking` and it uses `.network` files, one per interface. These two files configure the interfaces for my Internet-facing router:

`00-wan.network`

    [Match]
    Name=ens192
    
    [Network]
    DHCP=ipv4
    
    [DHCP]
    UseDNS=false
    UseDomains=false

`01-lab-transit.network`

    [Match]
    Name=ens224
    
    [Network]
    Address=10.0.250.1/24
    DNS=10.0.20.20
    DNS=10.0.20.21

<!--kg-card-end: markdown-->
## Dynamic routing
<!--kg-card-begin: markdown-->

To cover IBGP and static routing I installed FRR from its [own Debian repository](https://deb.frrouting.org/). By adding my user to the groups `frr` and `frrvty`, I am able to access its configuration in `/etc/frr/`. After enabling the BGP daemon, I took a first look at its configuration utlity `vtysh`. To my surprise the thing's almost exactly like Cisco's IOS with some few minor differences, so getting some static routes and internal BGP running was a breeze. Below is the FRR configuration from one of my new router VMs:

    frr version 7.1
    frr defaults traditional
    hostname VPN-R
    log syslog informational
    no ipv6 forwarding
    service integrated-vtysh-config
    !
    router bgp 4204206969
     bgp router-id 10.0.250.4
     neighbor 10.0.250.1 remote-as internal
     neighbor 10.0.250.2 remote-as internal
     neighbor 10.0.250.11 remote-as internal
     neighbor 10.0.250.12 remote-as internal
     !
     address-family ipv4 unicast
      network 10.0.60.0/24
     exit-address-family
    !
    line vty
    !

<!--kg-card-end: markdown-->
## Firewalling
<!--kg-card-begin: markdown-->

I'm not much of a fan of writing `iptables` commands by hand, which is why so far I've resorted to using utilities such as `ufw` and `firewalld` to configure firewalling. In a router though, neither of those aren't really suited out of the box for firewalling forwarded traffic. Recently there's been improvements on the `iptables` front with `nftables` that works as a replacement for the entire family of packet/frame handling utilities, including functionality from `iptables`, `arptables` and `ebtables` alike. As usual, Stretch has a pretty old version of it but stretch-backports has only a slightly older version compared to the latest in Buster and Sid. Grabbing `nftables` from there and digging through its documentation (and largely copying config other people pasted online) I came up with the following configuration file (`/etc/nftables.conf`) for my Internet-facing router:

    #!/usr/sbin/nft -f
    
    flush ruleset
    
    define internal = ens224
    define external = ens192
    
    table inet filter {
    	chain input {
    		type filter hook input priority 0;
    
    		ct state { established, related } accept
    		ct state invalid drop
    
    		iifname { lo, $internal } accept
    
    		ip protocol icmp accept
    
    		drop
    	}
    	chain forward {
    		type filter hook forward priority 0;
    
    		iifname { lo, $internal } accept
    		oifname $internal ct state { established, related } accept
    
    		oifname $internal tcp dport 443 accept
    		oifname $internal udp dport 50000-50999 accept
    		oifname $internal udp dport 1337 accept
    
    		drop
    	}
    	chain output {
    		type filter hook output priority 0;
    	}
    }
    
    table ip nat {
    	chain prerouting {
    		type nat hook prerouting priority 0
    
    		iif $external tcp dport 443 dnat 10.0.20.175
    		iif $external udp dport 50000-50999 dnat 10.0.20.178
    		iif $external udp dport 1337 dnat 10.0.250.4
    	}
    
    	chain postrouting {
    		type nat hook postrouting priority 0
    
    		oifname $external masquerade
    	}
    }

Reading this configuration is very straight-forward; define aliases for interfaces, define the input-, forwarding- and output-chains for the filter table, define pre- and postrouting for the NAT table. Rules are defined top-down as usual, allowing and blocking traffic as needed. Finally after checking the syntax for validity with `# nft -c -f /etc/nftables.conf` the firewall service is enabled and started, and can be reloaded mid-run if the configuration changes.

I can't speak on your behalf but at least to me, that is a million and then some times easier to reason about than a bunch of `iptables` commands strung together in a file (_cough_ `iptables-save` _cough_). And since it's still just netfilter in the background, the same filtering and NAT'ing principles apply; the configuration just doesn't want me throwing things (including myself) out the window.

## The rest

This setup doesn't unfortunately have a unifying configuration tool built-in (but I'm working on one), so for now Ansible is the best choice for external configuration. I haven't actually set it up for this yet, but knowing Ansible's flexibility and the available playbooks for [networking](https://github.com/aruhier/ansible-role-systemd-networkd), [FRR](https://github.com/mrlesmithjr/ansible-frr) and [nftables](https://github.com/ipr-cnrs/nftables) it'll be a breeze getting it going.

So that's how I got rid of VyOS for good.

### Edit (2019/11/24)

As I mentioned I've been working on such a unifying configuration tool, which is now [up in GitHub](https://github.com/Spanfile/Routing-Platform).

