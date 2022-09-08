---
layout: post
title: VyOS RADIUS user authentication
date: '2019-01-24 13:23:08'
tags:
- networking
---

Authenticating VyOS users with RADIUS is easy, but there's a couple of gotchas you should know of.

# How to actually do it

You can configure one or more RADIUS servers for authentication under `system login`:

    set system login radius-server 192.168.0.100 secret YourSharedSecretInPlainTextKeepItSafe
    set system login radius-server 192.168.0.100 timeout 3
    set system login radius-server 192.168.0.100 port 1812 # or whichever port you use

Additionally, you have to add all the users you wish to allow access to VyOS's own user database. They only need to be given the wanted access level, nothing else (not even a password).

    set system login user adminuser level admin

Now you can log in with `adminuser` with whatever password your RADIUS server is configured to use for it. If your RADIUS server is an AD-integrated NPS-server, leave any domain identification out and use an all-lowercase name (for example, the username for `AD\Administrator` would be `administrator`).

## Why the user though?

When VyOS is configured to use a RADIUS server for authentication, it'll always first try to authenticate given credentials through it. Next;

- If RADIUS declines the login, VyOS will try to authenticate a local user with the credentials.
- _Gotcha #1:_ If RADIUS accepts the login, VyOS will try to log in an user with the same name from its local users. The local user doesn't require a password here as it is only used when authenticating locally; the user only needs to exist so it can be logged in to.

Having the users locally as well is bit wonky but not too terrible. Unfortunately VyOS doesn't support users similar to LDAP or AD ones where they're created as needed, nor accepting a user's privilege level from RADIUS.

_Gotcha #2:_ This also means all local VyOS users are attempted to authenticate with RADIUS, even if the RADIUS server doesn't contain their credentials.

## What about security?

_Gotcha #3:_ VyOS only supports unencrypted PAP for transferring the credentials, so you'll want to secure the link between your VyOS and RADIUS server somehow if it's over an insecure network.

