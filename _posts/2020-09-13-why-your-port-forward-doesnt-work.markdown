---
layout: post
title: Why your port-forward doesn't work
date: '2020-09-13 21:40:17'
tags:
- networking
redirect_from: /why-your-port-forward-doesnt-work
---

There's a million and then some guides online that tell you how to set up a port-forward in your home router box, but what about when the forward doesn't actually work? In this post I've collected the common reasons why after a port-forward you still can't get your friend to connect to your cool Minecraft server (or for that matter, any online service behind your router box), and what you may be able to do about it.

I'll assume the port-forward is done correctly; the rules are correct, the addresses are fine, and that the service accepts incoming connections just fine.

## Reason #1: ISP traffic restrictions

The simple way to model your home network is as such:

![Simple home network diagram](/assets/2020/08/port-forward-1.png)

That cloud there is the Internet (through your ISP), then you got your router box to which all your devices are connected. In this scenario your router is plugged "directly" into the Internet and thus, is assigned an external address from the public address space (whether its v4 or v6 addresses).

To confirm the topology really is as described here, take note of your router's assigned external address. Then lookup "what is my ip" and the search engine (or its first result) tells you your publicly visible address - the address your connections are coming from and from which returning traffic gets back to you. If this address and your router's external address are the same, your network really looks like this one. If not, skip ahead to reason #2.

Given it has a public external address, anyone also connected to the Internet can send traffic to that address and it'll reach your router. Probably. It has to go through your ISP first of course, and they might not just let anything through.

From an ISP's point of view, home networks aren't meant to run any public services, because otherwise it'd count as a business and the ISP could bill you a _lot_ more for a business connection. Thus, since there's no need to allow any new connections in, they just outright block it all. But in today's world, having publicly accessible services doesn't mean you're hosting something. It could be VoIP, a game you play with friends, and so on. And so, ISPs do let connections in, but with some usual restrictions.

### Restriction #1: block "spammy" services

A large portion of the Internet's traffic is email, and a **large** portion of that is spam email. Consumer ISPs alleviate this issue by not allowing your network send or receive email directly with SMTP, thus blocking the ports 25, 465 and 587 on both directions. Your ISP might offer an SMTP relay, through which you can send and (maybe) receive email. So if you're trying to self-host email in your network, your ISP is aware how tremendously bad of an idea it is and doesn't let you use it in the first place.

### Restriction #2: block non-standard ports

For one reason or another, your ISP could be blocking ports that aren't widely associated with any known service. This basically means you can only get SSH in to port 22, HTTPS to 443 and so on. If you're using a non-standard port for your service, try its standard port.

### Restriction #3: block standard port

This is the opposite to the above. Instead of allowing services only in their standard ports, your ISP could _block_ them on their standard ports. It's kind of a good thing really; not having your SSH on 22 reduces the chance of an automated bot getting in when it can't quickly find you in the first place. Try some other port that isn't the service's standard port.

## Reason #2: double-NAT or CGNAT

Remember that network diagram from earlier? And the external address checking? It's very likely your network doesn't look like that, but instead looks like:

![Much more realistic network diagram](/assets/2020/08/port-forward-2.png)

The box on the left there is your ISP, from which you get to the Internet (the cloud). Now, instead of your router being plugged "directly" to the Internet, your ISP has another router doing NAT in between, in the same manner as your router does NAT. This is known as double-NAT or CGNAT (Carrier Grade NAT). The reason your ISP does this is because of IPv4 address space exhaustion. In the grand scheme of things there aren't that many v4 addresses, and they've all been allocated to some entities for years now. Effectively it means we've ran out of addresses.

What ISPs do to help with this issue is stick multiple customers behind a router that adds a layer of NAT. This allows the ISP to have hundreds (if not thousands) of customers behind just one public v4 address, instead of assigning each customer their own address (the previous scenario). This, however, means that there's no way you can get any new traffic into your network. None. Here, your router's external address is just some private address in your ISP's double-NAT'ed network, and your publicly visible address is that of the router's doing the double-NAT. Just like your router doing port-forwarding, in order for the new connections to reach you, they'd have to get through your ISP's double-NAT'ing router which you have no control over. And no, your ISP is not going to add a port-forward there for you.

Unfortunately there's not much you can do about this. You could contact your ISP and ask for a public address, but it's unlikely to happen. This kind of topology is very common in mobile connections, and sometimes on older DSL connections. Your best bet would be to get a proper broadband connection, of course researching beforehand if the connection plan gives you a public external address.
