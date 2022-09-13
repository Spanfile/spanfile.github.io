---
layout: post
title: I thought PiHole was kinda bad so I made my own
date: '2020-11-15 10:52:39'
tags:
- networking
- linux
redirect_from:
- /i-thought-pihole-was-kinda-bad-so-i-made-my-own
- /i-thought-pihole-was-kinda-bad-so-i-made-my-own/
---

TL;DR: it's a Rust binary called [Singularity](https://crates.io/crates/singularity) (because of the gravity lists, domain blackholing? it sounds cool don't judge). You give it sources for malicious domains and it outputs a Lua script that the PowerDNS Recursor uses to automatically respond with a null route (`0.0.0.0`) to all the malicious domains. No web UIs (SSH is enough), no `dnsmasq`, no system-overtaking installers, just a program that outputs a single file.

# What's this about PiHole?

I wanted network-wide ad blocking (and other known malicious domain blocking), which by now is arguably synonymous with [PiHole](https://pi-hole.net/). It's fine for everyday users, you just buy a Raspberry Pi, run one command and you're done. But I'm not an everyday user, so the amount of things it does and the amount of assumptions it makes aren't a great fit for my use.

- The installer "overtakes" the system it's being installed into and heavily assumes it's a Pi. Sure, yeah, the name has "Pi" in it but they're [officially supporting a bunch of others](https://docs.pi-hole.net/main/prerequisites/#supported-operating-systems) as well. 
- It wants to use a static address with `dhcpcd`. Sure, a Pi comes preinstalled with it. _But_, [they automatically install `dhcpcd` just to make sure it's there](https://docs.pi-hole.net/main/prerequisites/#ip-addressing), which _will_ cause conflicts with literally any other network management system. Again, a terrible thing for it do on anything but a Pi.
- One is none and two is one. DNS is built to be redundant, which in the case of local resolvers means running two of 'em (it's why all network configuration allow specifying two or more DNS resolvers). PiHole doesn't have anything to support running two (or more) of them in parallel, so two of them side-by-side are entirely separated from one another. It'd be quite cool if the fancypants web interface could aggregate statistics from both of 'em.
- They make modifying the `dnsmasq` conf- wait, no, they call it "FTL", which... eugh.
- They use their own fork of `dnsmasq` they call "FTL DNS" that lets them do blackholing more efficiently. They also market it as being [blazing fast](https://docs.pi-hole.net/ftldns/), and damn I sure hope so. It'd be pretty terrible if they'd managed to make their "faster than light" fork of `dnsmasq` slower than `dnsmasq` itself, which is already pretty fucking fast to begin with.
- They make modifying the actual `dnsmasq` configuration awkward, which is to say they don't really want you doing it at all.

So I didn't like PiHole for the short time I used it, so I thought I'd make my own.

# Enter PDNS

[PowerDNS Recursor](https://www.powerdns.com/recursor.html) is the high-end, easy-to-use, powerful-to-dig-into (pun intended) DNS recursor. It runs everywhere, and 'everywhere' includes a Pi. It offers a [Lua scripting interface](https://doc.powerdns.com/recursor/lua-scripting/index.html) that lets you program custom logic for responding to queries, such as responding with a `0.0.0.0` to queries for names specified in a list of malicious domains. Which is exactly what I made it do.

## The Lua interface

The Lua interface Recursor exposes has a [`preresolve()` function](https://doc.powerdns.com/recursor/lua-scripting/hooks.html#preresolve) that "is called before any DNS resolution is attempted, and if this function indicates it, it can supply a direct answer to the DNS query, overriding the internet". The interface also has so-called ["DNS suffix match groups"](https://doc.powerdns.com/recursor/lua-scripting/dnsname.html#dns-suffix-match-groups) that let you specify a collection of DNS names, and later match a given FQDN if it's equal to, or is a subdomain of any names in the collection (so if the collection has `spans.me`, in it, the name `blog.spans.me` would match). This kind of suffix match group used in the `preresolve()` function is how the blocking is done.

## How the blocking is done

We begin by specifying the malicious domain suffix match group and adding the unwanted domains. In Singularity's final output, this collection is _large_ (but that's fine)_._

    b = newDS()
    b:add{"malicious.domain", "facebook.com"}

Then, create the `preresolve()` function that returns a null route to all domains matched in the suffix group.

    function preresolve(q)
        -- if the queried name is in the suffix rgoup
        if b:check(q.qname) then
            -- since we're responding with an IPv4 address, respond only to A-queries
            if q.qtype==pdns.A then
                -- answer with an A-record pointing to the null route
                q:addAnswer(pdns.A, "0.0.0.0")
                -- tell Recursor we've handled this query and exit
                return true
            end
        end
        -- our matching didn't catch a malicious domain, tell Recursor we haven't touched this query and let it do it's thing
        return false
    end

Then... that's it. This script can then be saved somewhere and configured for Recursor's use with the [`lua-dns-script` config setting](https://doc.powerdns.com/recursor/settings.html?highlight=lua%20dns%20script#lua-dns-script). Now it's just a matter of maintaining the suffix match group, which is where Singularity steps in.

# Blackholing domains with Singularity

In its essence, [Singularity](https://crates.io/crates/singularity) reads known malicious domains from an URL, that can be either an HTTP/HTTPS URL, or a `file` URL for local files. It can read either [`hosts`-formatted domains](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts) (same format as in `/etc/hosts`, so `0.0.0.0 malicious.domain`) or just [plain domains, one-per-line](https://mirror1.malwaredomains.com/files/justdomains). It collects all the domains from the lists it's given, and finally outputs a complete Lua script as shown above. It can also output an `/etc/hosts`-style file, so it can be used anywhere where `/etc/hosts` can be used... which is everywhere really.

Its configuration allows specifying any combination of adlist inputs, and any combination of outputs. There's a couple of options per-input and per-output, which lets it be somewhat flexible in terms of what the input and outputs exactly contain.

## How I use it

In my Pi, I have PDNS Recursor running with a simple configuration:

<figure class="kg-card kg-code-card"><pre><code>config-dir=/etc/powerdns
include-dir=/etc/powerdns/recursor.d

allow-from=127.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16, 192.168.0.0/16, 172.16.0.0/12, ::1/128, fc00::/7, fe80::/10
local-address=0.0.0.0
local-port=553

forward-zones-file=/etc/powerdns/forward-zones
export-etc-hosts=/etc/powerdns/lan-hosts
export-etc-hosts-search-suffix=lan
lua-dns-script=/etc/powerdns/blackhole.lua

lua-config-file=/etc/powerdns/recursor.lua
hint-file=/usr/share/dns/root.hints
quiet=yes
security-poll-suffix=
setgid=pdns
setuid=pdns</code></pre>
<figcaption>/etc/powerdns/recursor.conf</figcaption></figure>

The `lua-dns-script` is the important bit. Singularity is configured to output its Lua script to that specified location. Its configuration looks as such:

<figure class="kg-card kg-code-card"><pre><code>[[adlist]]
source = "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts"

[[adlist]]
format = "domains"
source = "https://mirror1.malwaredomains.com/files/justdomains"

[[output]]
destination = "/etc/powerdns/blackhole.lua"
type = "pdns-lua"</code></pre>
<figcaption>$HOME/.config/singularity/singularity.toml</figcaption></figure>

Super simple; two adlists are specified, one in the `hosts`-format and the other in `domains`-format. One output is specified for the Lua script Recursor uses. Running Singularity with `sudo` (writing to the script's location requires root here):

    $ sudo -E ./singularity
    INFO Reading adlist from https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts...
    INFO Reading adlist from https://mirror1.malwaredomains.com/files/justdomains...
    WARN While reading https://mirror1.malwaredomains.com/files/justdomains, line #26407 ("") was parsed into an empty entry, so it was ignored
    INFO Read 84 671 domains from 2 source(s)

While running, it shows pretty progress bars for... progress. That one warning there is the result of a funny bug it had; if a line in the `domains`-format was empty, it'd be parsed as the catch-all `.` entry, which would match every domain, which meant blocking everything. Now such entries are ignored, and blocking works fine.

Now that Singularity has done its thing, Recursor can either be restarted or told to reload the Lua script with `sudo rec_control reload-lua-script`, and that's that. Recursor can be queried for names normally, except it'll return a `0.0.0.0` for everything malicious.

Getting Singularity automatically run every once in a while is easily done with standard tools like `cron` or `systemd` timers.

### "But wait, why does the Recursor config have local-port=553?"

Good question! Remember when I said "one is none and two is one"? I have two of these Pis running Recursor + Singularity, and on both there's `dnsdist`, PowerDNS's DNS load balancer software. `dnsdist` is the one listening on 53 on both Pis, and will forward queries to both the local Recursor and the opposing one from both.

<figure class="kg-card kg-code-card"><pre><code>newServer("192.168.0.3:553")
newServer("127.0.0.1:553")
setLocal("0.0.0.0")</code></pre>
<figcaption>/etc/dnsdist/dnsdist.conf</figcaption></figure>

Then for all my network clients, I have both Pis set as nameservers, so there's redundancy and load balancing going on.

## But is it "blazing fast" or "faster than light"?

I dunno, I'm not well-versed with marketing jargon ¯\_(ツ)\_/¯

It works fine for my use, and I haven't noticed it being significantly slower than just normal recursion. Recursor seems to support pre-compiled Lua scripts that might speed up the resolving override, but I haven't tried that out yet.

## What about the fancy web UI? Or metrics?

I don't care for web UIs for my DNS resolvers, I have SSH. Recursor's Lua scripts [support outputting metrics](https://doc.powerdns.com/recursor/lua-scripting/statistics.html#generating-metrics) which I'm using to output a metric called "blocked-queries" that ends up among all the other metrics Recursor outputs. It is incremented for each blocked query.

Other than that, since it's just Recursor and dnsdist running, they can be monitored with whatever supports monitoring them. Go wild.

