---
layout: post
title: LXC part 1
tags:
    - lxc
    - homelab
---

This is part #1 of #however-many posts where I explain how I use LXC containers to run my network services. It's a lot of stuff to cover, so I'm splitting it into multiple posts, each covering a somewhat cohesive topic.

Read the other posts here:

-   epic
-   list
-   of
-   posts
-   here

# Briefing

I have some SFF PCs that I use to host various local network services. In most cases the hosts would be hypervisors running virtual machines, or application container hosts with Docker or Podman (maybe someone mad enough even uses Kubernetes). When I first got the machines, I considered using them as hypervisors with KVM. I had used Proxmox before, and thought I'd roll my own solution this time. It would allow me more freedom and control over the solution, not being tied to a single product. That solution would've likely been a Fedora host that runs virtual machines with libvirt and QEMU, however, I had a thought:

> If I'm going to run Linux to run virtual machines that each run more of the same Linux, why not cut the virtualisation layer, just run one Linux and use containers?

## The thought

Linux containers generically refer to separate userspaces running on the same kernel. This is achieved with cgroups, network namespaces, drive overlays and whatnot. The most common place you run into containers is Docker and the likes; application containers. Each container runs only a single application, such as a database, a web server, yeah you get it this is beginning to sound like an SEO boost for modern devops practices.

LXC, while could be used for the same purpose, is more suited for running an entirely separate system inside the container. In fact, LXC containers are often referred to as system container. What this means in practice is the one application the container runs is `/sbin/init`, in this case systemd. It then behaves like it usually does, setting up networking, starting services such as an SSH server and whatnot. Just like it would on a normal physical host.

By attaching the containers to a network bridge, they get direct access to the local network, can ask for an address with DHCP, so now since they're each running a Linux system, they look like every other host from the outside. Just like virtual machines would, except these don't have the virtualisation overhead. There are of course drawbacks and challenges to it, but that just makes it more fun. We'll get to those as the story goes along.

# The plan

Here're the top-level guidelines that I try to follow with the project:

-   **Automate everything**; in practice I'm only using Ansible, I did consider adding Terraform but it has nonexistent support for the lower-level LXC containers. This extends to the practice known as _IaC_; Infrastructure-as-Code. I have one git repository with a whole bunch of Ansible in it that acts as a source of truth for the network. Any modifications to the network happen from here.

-   **Make everything disposable**; hosts, containers, should be instantly disposable and easily recreated. All important data is stored in a dataset external to the container and backed up. Hosts and containers should be easily destroyed and recreated without data loss. As how things currently are, the physical hosts aren't as disposable as they could be. Some of you might already have ideas how to achieve that; I'll get to it.

-   **Some epic third thing**; to make the list look better.

# Part 1: LXC hosts

Hardware-wise the SFF PCs I'm using have 6th-generation Skylake CPUs, varying amounts of memory, SSDs, and quite importantly half-height PCI-e slots. And, best of all, they have actual DB9 serial ports on the back. Real _serial ports_!

Okay to be fair, motherboards still have COM headers on them and you can get PCI-e brackets that adapt that header to a DB9 serial port, but these have actual ones, on the back of the motherboard! Right next to the DisplayPort!

---

### Tangent #1: serial ports

Serial ports are still very useful with server and network hardware for local configuration. Since every good server runs Linux and every good network device has a CLI, every good device is configured with a command line; i.e. text. Most of the time you control them remotely with SSH, but for a local Linux server you'd need a monitor to see it and a keyboard to control it (unless it's a server and you can use the out-of-band management controller). But if it's just a text console, why do you need an entire monitor and keyboard when serial is right there? The RS232 serial protocol is just text anyways, so just stick the console in there (or to be more precise, run `serial-getty` on `ttyS0`) and _bam_, you have console access to your Linux server with just a cable and a laptop. No need to figure out how to power an entire monitor.

---

I've picked out two hosts that both have i5-6500 CPUs, one has eight gigabytes of memory and the other 16, both have one terabyte SSDs, and additional PCI-e NICs. I'm using Intel I350-T4 four-port RJ45 gigabit NICs, however I'm planning to upgrade these to 10 gigabit SFP+ NICs later, probably Mellanox ConnectX-3s.

explain OS and go from there
