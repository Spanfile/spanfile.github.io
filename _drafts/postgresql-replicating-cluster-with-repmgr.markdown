---
layout: post
title: PostgreSQL replicating cluster with repmgr
---

I have an existing PostgreSQL 11 server functioning as a central database for my various services. It's just a single server though and has essentially no redundancy, only a backup. To fix that I looked into adding redundancy into it and this is what I ended up doing.

## Options

Arguably the thing seasoned database admins think of when they think of PostgreSQL redundancy is [pgpool2](https://www.pgpool.net/mediawiki/index.php/Main_Page). It's a powerful Postgres "middleman" that introduces load balancing, high availability and sharding for one, but it's also notorious for being difficult to configure for more complex environments. Another somewhat more recent option is [repmgr](https://repmgr.org/) (short for replication manager) that introduces replication and failover into a cluster of PostgreSQL servers. It doesn't do as much as pgpool2 would, but it's a lot simpler to set up and configure. I chose to use it here.

## Primary and stanby setup
<!--kg-card-begin: markdown-->

First things first, the existing Postgres (which will act as the primary) has to be upgraded to 12. It runs on Debian and is acquired through Postgres' APT-repository, so installing 12 is just a matter of `# apt install postgresql-12`. Looking up a guide online I found [this one](https://dev.to/jkostolansky/how-to-upgrade-postgresql-from-11-to-12-2la6) and to begin, I copied the old configuration files `postgresql.conf` and `pg_hba.conf` into the new installation.

    # cp /etc/postgresql/11/main/postgresql.conf /etc/postgresql/12/main/postgresql.conf
    # cp /etc/postgresql/11/main/pg_hba.conf /etc/postgresql/12/main/pg_hba.conf

I stopped the running PostgreSQL instance (migration isn't as live as one would hope) and changed user to the `postgres` user.

    # systemctl stop postgresql
    # su postgres

PostgreSQL comes packed with simple tools to perform the migration, the first one being `pg_upgrade`. This command moves the existing server into the new one - but not yet doing it because of the `--check` flag, only ensuring the migration can be done safely.

    $ /usr/lib/postgresql/12/bin/pg_upgrade \
      --old-datadir=/var/lib/postgresql/11/main \
      --new-datadir=/var/lib/postgresql/12/main \
      --old-bindir=/usr/lib/postgresql/11/bin \
      --new-bindir=/usr/lib/postgresql/12/bin \
      --old-options '-c config_file=/etc/postgresql/11/main/postgresql.conf' \
      --new-options '-c config_file=/etc/postgresql/12/main/postgresql.conf' \
      --check

It indicated the migration is safe so I ran the same command again, this time without the `--check` flag to actually perform the migration. It dropped two scripts into the current directory; `analyze_new_cluster.sh` and `delete_old_cluster.sh`. These are used to finalise the migration. Before that, the new server has to be started.

    # systemctl start postgresql@12

Changing into the `postgres` user again I ran the `analyze_new_cluster.sh` script.

    # su postgres
    $ ./analyze_new_cluster.sh

This script indicated that the new server is running normally and that the migration succeeded. Now it's safe to completely remove the old Postgres 11.

    # apt purge postgresql-11*
    # rm -rf /etc/postgresql/11

And finally remove the old server data with the given script.

    # su postgres
    $ ./delete_old_cluster.sh

The server is now running PostgreSQL version 12. Now to get the standby server going.

<!--kg-card-end: markdown-->
### Preparing the standby
<!--kg-card-begin: markdown-->

I started off with a fresh Debian 10 installation and added Postgres' APT-repository and signing key, and then installed PostgreSQL 12.

    # nano /etc/apt/sources.list.d/pgdg.list
    deb http://apt.postgresql.org/pub/repos/apt/ buster-pgdg main
    # wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
    # apt update
    # apt install postgresql-12

After installation, the user `postgres` is created and its home directory is `/var/lib/postgres`. This directory contains PostgreSQL's data directory `12/main` as well, which is important later on.

<!--kg-card-end: markdown-->
## repmgr

### Installation
<!--kg-card-begin: markdown-->

`repmgr` is provided through [2ndQuadrant's](https://dl.2ndquadrant.com/default/release/site/) apt repository and can be added to a Debian system with their [installation script](https://dl.2ndquadrant.com/default/release/get/deb) and the latest version of `repmgr` suitable for the system's PostgreSQL can be installed from it:

    # curl https://dl.2ndquadrant.com/default/release/get/deb | bash
    # apt install postgresql-12-repmgr

Piping scripts from the Internet straight into bash isn't always a great idea so do take a look at the script beforehand to know what it's doing. TL;DR it possibly installs `lsb-release` and `apt-transport-https` if they're not already installed, adds 2ndQuadrant's apt signing key and finally adds their repo.

<!--kg-card-end: markdown-->
### PostgreSQL configuration
<!--kg-card-begin: markdown-->

Out of the box PostgreSQL 12 has suitable configuration options set for the write-ahead log (WAL) and replication, but a couple archiving options need to be tweaked in `/etc/postgresql/12/main/postgresql.conf`. On both nodes:

    archive_mode = on
    archive_command = '/bin/true'

Authentication options need to be tweaked in `/etc/postgresql/12/main/pg_hba.conf` in order for `repmgr` to be able to access the PostgreSQL instances. In addition to what is already there, append to both nodes (replace `network` with the local network both nodes are in):

    local replication repmgr trust
    host replication repmgr 127.0.0.1/32 trust
    host replication repmgr <network>/24 trust
    
    local repmgr repmgr trust
    host repmgr repmgr 127.0.0.1/32 trust
    host repmgr repmgr <network>/24 trust

Create an user and a database for `repmgr` on the primary node. The system user `postgres` is by default allowed to manage the instance, switch to it.

    # su postgres
    $ createuser -s repmgr
    $ createdb repmgr -O repmgr
    $ psql
    postgres=# ALTER USER repmgr SET search_path TO repmgr, "$user", public;

Restart the instances on both nodes to apply the changes:

    # systemctl restart postgresql

Check that the standby can access the primary instance:

    # psql 'host=10.0.20.60 user=repmgr dbname=repmgr connect_timeout=2'

<!--kg-card-end: markdown-->
### repmgr configuration
<!--kg-card-begin: markdown-->

`repmgr` itself requires very little configuration initially. Change to the system's `postgres` user and create `repmgr`'s configuration file to the `postgres` user's home directory. This configuration tells `repmgr` the current node's information required for replication:

    # su postgres
    $ cd
    $ vim repmgr.conf
    node_id=1
    node_name='node1'
    conninfo='host=10.0.20.60 user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/postgresql/12/main'

It's important to specify the data directory correct here; `repmgr`'s documentation specifies a different path to what Debian's PostgreSQL 12 uses.

<!--kg-card-end: markdown-->
### Primary server registration
<!--kg-card-begin: markdown-->

`repmgr` is now ready to initialise by registering the first primary server. While still using the `postgres` user and in their home directory, run `repmgr` and register the current server as primary:

    $ repmgr primary register

The `repmgr` utility will read the configuration file from the current directory, but you can specify it separately with the `-f` flag.

<!--kg-card-end: markdown-->
### Standby server cloning
<!--kg-card-begin: markdown-->

The standby server is "cloned" from the primary with `repmgr`. Like with the primary, change to the `postgres` user and create a configuration file for `repmgr`. Specify the correct values in the configuration to reflect the standby:

    node_id=2
    node_name='node2'
    conninfo='host=10.0.20.61 user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/postgresql/12/main'

Use `repmgr` to clone the primary server's data directory to the standby:

    $ repmgr -h 10.0.20.60 -U repmgr -d repmgr standby clone

The standby might already have a populated data directory, in which case shut down the PostgreSQL instance and force the clone with the `--force` flag.

<!--kg-card-end: markdown-->