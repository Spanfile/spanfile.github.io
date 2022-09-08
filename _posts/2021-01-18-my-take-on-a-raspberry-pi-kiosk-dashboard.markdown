---
layout: post
title: My take on a Raspberry Pi kiosk dashboard
date: '2021-01-18 18:53:26'
tags:
- homelab
- linux
---

Like everyone and their mother-in-law, I have a local Grafana instance with a bunch of dashboards filled with graphs and bar gauges about things in my network. I have a [desktop network rack](/6u-desktop-network-rack-build/) with space on the top for a spare 24" monitor I have, so I thought I'd make a "24/7" Grafana kiosk display on it with a Raspberry Pi.

I'll be using standard Xorg and Chromium, and X11VNC for remote control. They'll all be controlled and ran with systemd user-units for the `pi` user. My reason for the separate services is that many other guides online for a similar kiosk run Xorg and Chromium directly from `.bashrc` or the like, which wouldn't allow restarting Chromium itself very easily, which is essential especially during testing the setup. Using separate services for Xorg and Chromium lets me restart either at will without having to reboot the entire Pi.

## The hardware

Super simple stuff. As hinted by the network rack build post, there's a Raspberry Pi model 3B in the rack, powered via Power-over-Ethernet. This Pi has been a sort-of catch-all for miscellanous things I've done, so this time it gets to run the kiosk. The monitor I'm using is some Asus 24" 1080P-monitor with an HDMI input, which I've place on top of the network rack and connected to the Pi via HDMI.

## The software

The interesting bit. I started off with a stock Raspberry Pi OS Lite and set up some basics with the `raspi-config` tool. Most importantly, I enabled SSH and console autologin. I left its VNC out, since it installs RealVNC which is a bit much for this need. I'll be setting up X11VNC later on. I installed Xorg, Chromium and some accompanying software in order to get the dashboard displayed first and foremost.

    # apt install chromium-browser openbox unclutter xserver-xorg xinit x11-xserver-utils x11vnc

### Dashboard pre-requisites

Sourcing from a bunch of places online, I scratched together a pre-script that sets up some things for Xorg and Chromium.

`~/dashboard-pre.sh`:

    #!/bin/bash
    sleep 1
    xset -dpms
    xset s off
    xset s noblank
    
    sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' /home/pi/.config/chromium/Default/Preferences
    sed -i 's/"exit_type":"Crashed"/"exit_type":"Normal"/' /home/pi/.config/chromium/Default/Preferences
    unclutter &

In my testing I found that the upcoming systemd unit would run too soon after Xorg had started, so the `xset` commands would freeze. I'm unsure if I could fix the service itself, but a simple one second sleep did the trick here.

Then, the services to run after the user logs in, which happens automatically since I previously set the console to autologin to the `pi` user.

### Xorg services

Xorg requires a service and a socket definition.

`~/.config/systemd/user/xorg@.socket`:

    [Unit]
    Description=Socket for xorg at display %i
    
    [Socket]
    ListenStream=/tmp/.X11-unix/X%i
    
    [Install]
    WantedBy=default.target

This socket is responsible for maintaining a communication socket for any Xorg instances, as denoted by the systemd unit index `%i` which represents the display number. The `:0` display is `xorg@0.socket` and so on. Since this is a socket service, when any communication comes into the socket, systemd will automatically start the corresponding service.

`~/.config/systemd/user/xorg@.service`:

    [Unit]
    Description=Xorg server at display %i
    Requires=xorg@%i.socket
    After=xorg@%i.socket
    
    [Service]
    Type=simple
    SuccessExitStatus=0 1
    ExecStart=/usr/bin/startx -- :%i

The actual service for Xorg is simple (pun intended). It has display indexing just like the socket, and requires its corresponding socket to be running before it can run itself. It simply calls `startx` with the configured display number, which in turn runs Openbox, the window manager. This service does _not_ have an `[Install]` section, since the socket is responsible for starting the service, instead of the service starting itself when the user logs in.

### Dashboard service

Just Xorg and Openbox by themselves won't get anything on the screen, so next up is the service for the Chromium kiosk itself.

`~/.config/systemd/user/dashboard.service`:

    [Unit]
    Description=Grafana dashboard
    After=xorg@0.service
    Requires=xorg@0.service
    
    [Service]
    Environment=DISPLAY=:0.0
    Environment=XAUTHORITY=/home/pi/.Xauthority
    Type=simple
    ExecStartPre=/home/pi/dashboard-pre.sh
    ExecStart=/usr/bin/chromium-browser --kiosk --incognito --window-position=0,0 --noerrdialogs --disable-infobars "http://grafana.lan:3000/playlists/play/1?kiosk"
    Restart=on-abort
    
    [Install]
    WantedBy=default.target

A bit more complex. Like Xorg depends on its socket, this service depends on Xorg running at display 0 (thus `xorg@0.service`). The service sets the environment variables required for applications running on Xorg, runs the `dashboard-pre.sh` script from before as a pre-run and actually runs Chromium with the appropriate parameters for running it as a kiosk. The final parameter specifies which site it'll open, which in this case is the local Grafana instance and a playlist within it.

The dashboard itself is now ready to run with systemctl.

    $ systemctl --user daemon-reload
    $ systemctl --user enable --now xorg@0.socket
    $ systemctl --user enable --now dashboard.service

As explained earlier, only the socket for the Xorg server at display 0 is enabled and started, and the dashboard is started as well. This runs Xorg and opens Chromium on the display connected to the Pi, which opens the Grafana playlist automatically.

### X11VNC

X11VNC is set up in a similar manner with a user systemd service. To be just that one tiny bit more secure (and to stop X11VNC from nagging about it), I created a VNC password for myself with `x11vnc -storepasswd`. Then the service.

`~/.config/systemd/user/x11vnc.service`:

    [Unit]
    Description=X11VNC server
    After=xorg@0.service
    Requires=xorg@0.service
    
    [Service]
    Environment=DISPLAY=:0.0
    Environment=XAUTHORITY=/home/pi/.Xauthority
    ExecStart=/usr/bin/x11vnc -usepw -display $DISPLAY -forever -ncache 10
    
    [Install]
    WantedBy=default.target

Just like the dashboard service, this service depends on the Xorg server service running on display 0. It also sets the required environment variables, and runs X11VNC on the configured display. Importantly it uses the `-forever` option, which keeps it running after a connected client disconnects. Normally it'd shut down, which isn't what I want.

Much like the other services, it too is set to run on user login.

    $ systemctl --user daemon-reload
    $ systemctl --user enable --now x11vnc.service

## Final words

If everything works correct, the Pi should be safe to reboot and it'll automatically log in after booting, run Xorg and open Chromium to the Grafana dashboard playlist.

If you're interested, my two Pis run PowerDNS Recursor with a custom tool I developed called [Singularity](https://github.com/Spanfile/Singularity), which configures Recursor to reply with a null route to known malicious domains. I've [written about it here](/i-thought-pihole-was-kinda-bad-so-i-made-my-own/). The tool exports a `blocked-domains` stat among the other Recursor stats, which the DNS dashboard displays.

