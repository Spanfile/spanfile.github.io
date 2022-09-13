---
layout: post
title: My DN42 network (so far)
date: '2021-12-13 12:47:21'
tags:
- networking
- homelab
redirect_from:
- /my-dn42-network-so-far
- /my-dn42-network-so-far/
---

A couple months ago I joined [DN42](https://dn42.us/), and I've spent some of my spare time since then building my little chunk of (mostly) virtualised network there.

If you haven't heard of DN42 before, their site says it's a "dynamic interconnected VPN", and that is correct, but I don't think it tells the whole story. I'd instead describe it as a virtual internet on top of the real one. It functions more-or-less like the actual internet, just without so much bureaucracy and physicality. Instead of many entities building physical networks and gathering in the same datacenters - IXes - there are hobbyists building their own networks and using the actual internet as one large IX. Networks aren't connected with physical cables linking routers (most of the time), but instead are connected with VPN tunnels.

I've hid a certain theme in the network, see if you can figure it out! Shouldn't be too difficult ;)

# The network

This network diagram is an SVG, feel free to open it in a new tab to get a clearer view.

![The network diagram](/assets/2021/12/Spans-DN42.svg)

Let's start from the beginning. I want to use the network to learn about company networking; different interconnected sites, the various internal services, public services and so on. I picked the AS number 4242422038. I've been allocated (well, I chose to allocate myself) an IPv4 block, 172.23.38.128/27 and an IPv6 block fd38:119:314::/48. I picked a scheme that lets me split the space into various sites and their subnets. IPv4 allocations from the public block are more-or-less arbitrary, since it's so small. Right now I've just split it into two /28s. If I need more in the future, I might just request another /27 instead of splitting these further into even smaller blocks.

For the private networks I chose the block 10.42.0.0/16, out of which it is simple to allocate individual /24s - the realistic minimum for a private IPv4 network. This scheme lets me pick anything between 0 and 255 for the third octet, so I went with using the site identifier for it. This by itself means each site can only have a single /24, but this is my network, I can just pick some unused subnet and assign it to a site that needs one.

The v6 block is easier, since there's a lot more space for separate subnets in the single allocation. Since the realistic subnet allocation for IPv6 is a /64, a /48 leaves 16 bits, or two bytes, of address to pick for each subnet. In total that's 65,536 subnets, which should be enough ;) I split the two bytes between /48 and /64 into two, where the upper byte is the site identifier and the lower byte is the subnet identifier. The zeroth subnet is always the public segment for each site.

In practice, this means the first site, also known as "Kludge", gets 172.23.38.128/28 & fd38:119:314::0/64 for the public segment, and 10.42.0.0/24 & fd38:119:314:1::/64 for the private segment. The second site, "Jank", gets 172.23.38.144/28 & fd38:119:314:100::/64 for its public segment and 10.42.1.0/24 and fd38:119:314:101::/64 for its private segment.

# The sites

The routers in each site are always assigned the first address in each subnet. The addresses assigned in the public segment are also what they use as router identifiers for (i)BGP and point-to-point tunnel endpoints.

Each site at a minimum has:

- A pair of local DNS resolvers. These provide a local DNS service for the site. They use DN42's anycast resolvers for upstream DNS.
- A monitoring satellite. Except of course in the primary site, Kludge, which has the monitoring master. These satellites help ease the load on the site-to-site connectivity and monitoring by delegating host and service checking to them, and reporting their statuses back to the master.

Most services across the sites are running in separate virtual machines hosted by my vSphere lab cluster. The VMs are running AlmaLinux 8.

## Kludge

The primary site is Kludge, and it hosts most of the services in the network. The site publicly hosts:

- One of the two nameservers.
- The public dashboard to view network statistics.
- Soon: a looking glass service.

Privately it hosts:

- The hidden primary nameserver. More about this later.
- A Prometheus instance for gathering metrics. This is used by the public dashboard.
- An Icinga2 master instance for monitoring everything across the sites. More about this later as well.

Its router is FI-TRE1, a virtual machine. Its denotes where the router is physically located in the world. This router is connected to various peers in the DN42 network with Wireguard, so it also handles all connectivity to DN42 from my network. For BGP and other routing it uses Bird2, and for IPsec it uses strongSwan with swanctl.

## Jank

The first secondary site is called Jank. Publicly it only hosts the second nameserver. Privately it has an Icinga2 satellite host for monitoring its services. Its router is FI-TRE2, and it's a physical Juniper SRX300 router. It connects to only FI-TRE1 with a point-to-point IPsec tunnel. Not many people seem to want to do IPsec in DN42, I wonder why.. ;)

Physically this site runs in the very same cluster as the first site, it's separated only logically with VLANs.

# The services

## Nameservers

My domain name in DN42 is spans.dn42. I host two authoritative nameservers for it, nicknamed Unix and Epoch. The first one is hosted in the site Kludge, and the other one in the site Jank. I employ a hidden master technique for them, which means both of the public nameservers are actually secondary nameservers. The primary nameserver lives isolated in the private segment in Kludge, so exposure to it from the public network is limited. It cannot even be queried directly; its only purpose is to host the various zones and synchronise them to the two public secondaries. The secondaries don't have the right to modify the zones, so in the event of their breach (who would be so vile in DN42 anyways..) an attacker wouldn't have control of the zones.

Each of the nameservers run PowerDNS with the PostgreSQL backend. The hidden primary uses PowerDNS-Admin to modify and control the zones with a web interface.

The private resolvers in each site run PowerDNS's Recursor.

## Monitoring and statistics

In Kludge, there is a centralised Icinga2 instance that acts as a master for all monitoring zones. In each site other than Kludge, there is an Icinga2 satellite host that defines the monitoring zone for that site. Icinga delegates checking for hosts and services in the sites to their respective, and the satellites then report the results back to the Icinga master. This eases the load between the sites, as the master doesn't have to check each host and service in the sites itself.

For each Linux host, the basic things are monitored; system load, memory use, drive usage. Additionally, for each host their own services are monitored, such as DNS availability for each resolver, and authoritative zone synchronisation between the primary and two secondary nameservers. The actual monitoring happens using the SSH daemon in each host, such that the monitoring instance opens an SSH connection to each host, runs the corresponding plugin and returns its output. This removes the need for separate monitoring agents.

In Kludge, there is a Prometheus instance that gathers metrics from various hosts in the network. These include:

- Routers in each site. Depending on their type the collection method varies. VM routers use node\_exporter just like any other VM, wireguard\_exporter for Wireguard metrics and bird\_exporter for routing metrics. Physical routers are queried with SNMP.
- The authoritative PowerDNS server and the PowerDNS Recursor both expose metrics related to their DNS operation. These include incoming queries, answered questions, cache hits, answer latency and so on.

The [public dashboard](https://dash.spans.dn42) displays some of these metrics. It uses a self-signed certificate because I haven't yet been arsed enough to get a certificate signed by the DN42 CA.

# Final words

It's been quite fun so far messing with the network and its various services in a lab environment. I'm still planning to add more things such as centralised authentication, virtual workstations in the zones, more zones in separate physical locations, and a looking glass I'm writing myself - more about that later? ;)

Oh, did you figure out the hidden theme?

