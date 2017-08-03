# HOW TO: Best practise Seafile Server manual
This manual will show how to setup a Seafile Server using MariaDB, Memcached and nginx as a reverse proxy on Debian operating system.

[TOC]

All operations will be peformed as root unless otherwise specified. So login as root or 'su' to root, if logged in as ordinary user:
```bash
su
```

## Install Operating system
In this manual we will use Debian 9 *"Stretch"* 64-bit as operating system. It can be obtained from [https://www.debian.org/](https://www.debian.org/). The "64-bit PC Network installer" will be sufficiant.
In this manual we will put the seafile installation and configuration in `/opt` and seafile_data in `/srv`. If you devide your disk(s) into several partitions, 10 GiB will be enough for `/`. If you have up to 2 GiB of memory, choose 2 GiB for swap, otherwise 1 GiB will be sufficiant. `/srv` will contain all data in seafile server and needs to be at least that size plus some spare for deleted files.

## Provide a static IPv4 address
If you have a root server or vServer or for whatever reason you got a static IPv4 address for your server, just use that address and your are done with this chapter. If you have such a server with IPv6 only, read the **No IPv4 ** at the End of this chapter.

After installing the operating system it often gets its IPv4 adress via DHCP. To avoid trouble in the future this address should be static in most cases.
```sh
root@cloudserver:~# ip route
default via 192.168.1.1 dev ens3 
192.168.1.0/24 dev ens3 proto kernel scope link src 192.168.1.22 
```

The `default` line tells us 192.168.1.1 is the default gateway, ens3 is the network device in our case. We need to remember that.
```sh
root@cloudserver:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:ae:a4:12 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.22/24 brd 192.168.1.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:feae:a412/64 scope link 
       valid_lft forever preferred_lft forever
```

The interresting part is the inet line in our network device (ens3). It tells us 192.168.1.22 is the current IPv4 address, /24 is the network mask in CIDR notation, which is 255.255.255.0 as subnet mask (see [https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)) and 192.168.1.255 is the broadcast address.

We need to pick an unused IPv4 address, which is not within the dhcp range. Mostly the dhcp server is in the router. You will find the description there. x.x.x.1 ist the gateway in this case as we have seen. dhcp range starts normaly at x.x.x.20 or above. x.x.x.2 is normally free, but you should be sure about that to prevent annoying network problems. I choose 192.168.1.2 as static IPv4 address.

```sh
root@cloudserver:~# cat /etc/resolv.conf
domain fritz.box
search fritz.box
nameserver 192.168.1.1
root@cloudserver:~# hostname -f
cloudserver.local
```

We take the dns-server (= nameserver)  from here. In this case the dhcp server put this computer in the domain 'fritz.box' while I put it into the domain 'local'. Pros and cons what to take are not scope of this how-to. I will leave it as it is for now.

Now it is better to have a backup of the configuration. We can put it in our home directory:
```sh
root@cloudserver:~# cp /etc/network/interfaces ~/
```

This is the actual network configuration:
```sh
root@cloudserver:~# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens3
iface ens3 inet dhcp
# This is an autoconfigured IPv6 interface
iface ens3 inet6 auto
```

Change the network configuration to your actual values (... means no change in there).
```
...

# The primary network interface
allow-hotplug ens3
iface ens3 inet static
  address 192.168.1.2
  netmask 255.255.255.0
  gateway 192.168.1.1
  dns-domain local
  dns nameservers 192.168.1.1
# This is an autoconfigured IPv6 interface
iface ens3 inet6 auto
```

Reboot and good luck. Test your network configuration to assure it's working as desired. `ip addr` should show your static IPv4 address, `ip route` still the gateway and so on. If something does not work, you have a file named "interfaces" in the home directory of user root, that is your backup. You can copy it back to `/etc/network/interfaces` and start all over.

### No IPv4
Having a root server or a vServer without any IPv4 at all is not really within the scope of this manual. Probably the best way is to get a domain now and point it to your IPv6 address.

### Literature
[Debian Reference Manual: The network interface with the static IP](https://www.debian.org/doc/manuals/debian-reference/ch05.en.html#_the_network_interface_with_the_static_ip)

## Diagnostic Tools
Install curl
```bash
apt-get install curl
```
curl is a tool to transfer data from or to a server. We will use it to diagnose the web server.

Install nmap
```bash
root@cloudserver:~# apt-get install nmap
```
nmap is a network exploration tool. We will use it as a port scanner to diagnose running services.

## Install Seafile Server
We will install Seafile Server in `/opt/Seafile/Server`, Seafile data in `/srv/Seafile/seafile_data`.
### Prerequisites
Install required packages for Seafile Server:
```sh
root@cloudserver:~# apt-get install python-setuptools python-imaging python-ldap python-mysqldb python-memcache python-urllib3 memcached
```

Install MariaDB:
```sh
root@cloudserver:~# apt-get install mariadb-server
```

Create a user and his own group to run the Seafile Server. We choose the name 'seafserver' for the user as well as for his group. `useradd -h` for a short help of the switches.
```sh
root@cloudserver:~# mkdir /opt/Seafile
root@cloudserver:~# useradd -U -m -d /opt/Seafile/Server seafserver
```

You can check the result:
```sh
root@cloudserver:~#  ls -l /opt/Seafile/
total 4
drwxr-xr-x 2 seafserver seafserver 4096 Jul  3 17:10 Server
root@cloudserver:~#  grep seafserver /etc/passwd
seafserver:x:1001:1001::/opt/Seafile/Server:
root@cloudserver:~#  grep seafserver /etc/group
seafserver:x:1001:
```

Download the lastest Seafile Server package from [here](https://www.seafile.com/en/download "here") and put it in `/opt/Seafile/Server/installed`. Adjust the version number.
```sh
root@cloudserver:~#  mkdir /opt/Seafile/Server/installed
root@cloudserver:~#  wget -P /opt/Seafile/Server/installed https://download.seadrive.org/seafile-server_6.1.1_x86-64.tar.gz
```

Install Seafile Server:
```sh
root@cloudserver:~# tar -xz -C /opt/Seafile/Server -f /opt/Seafile/Server/installed/seafile-server_*
```

It should look something like this:
```sh
root@cloudserver:~# ls -l /opt/Seafile/Server
total 8
drwxr-xr-x 2 root root 4096 Jul  3 17:22 installed
drwxrwxr-x 6  500  500 4096 Jun 13 07:52 seafile-server-6.1.1
```

Configure Seafile Server and databases.
```sh
root@cloudserver:~# mkdir /srv/Seafile
root@cloudserver:~# /opt/Seafile/Server/seafile-server-*/setup-seafile-mysql.sh
```

- [ server name ] Cloud (whatever you like)
- [ This server's ip or domain ] 192.168.1.2 (Server's IP address)
- [ default "/opt/Seafile/Server/seafile-data" ] /srv/Seafile/seafile-data
- [ default "8082" ] (leave the port as it is)
- [ 1 or 2 ] 1 (create new databases)
- [ default "localhost" ] (database runs on this server)
- [ default "3306" ] (standard port for mysql or mariadb)
- [ root password ] <enter root password>
- [ default "seafile" ] (it's the name of the user in mariadb)
- [ password for seafile ] (give the user a password, no need to remember)
- [ default "ccnet-db" ]
- [ default "seafile-db" ]
- [ default "seahub-db" ]

Now the user seafserver needs to own the whole stuff:
```sh
root@cloudserver:~# chown -R seafserver:seafserver /opt/Seafile/Server  /srv/Seafile
```

It should look like this:
```sh
root@cloudserver:~# ls -l /opt/Seafile/Server
total 20
drwx------ 2 seafserver seafserver 4096 Jul  3 17:59 ccnet
drwx------ 2 seafserver seafserver 4096 Jul  3 17:59 conf
drwxr-xr-x 2 seafserver seafserver 4096 Jul  3 17:22 installed
drwxrwxr-x 6 seafserver seafserver 4096 Jun 13 07:52 seafile-server-6.1.1
lrwxrwxrwx 1 seafserver seafserver   20 Jul  3 17:59 seafile-server-latest -> seafile-server-6.1.1
drwxr-xr-x 3 seafserver seafserver 4096 Jul  3 17:59 seahub-data
root@cloudserver:~# ls -l /srv/Seafile
total 4
drwx------ 3 seafserver seafserver 4096 Jul  3 17:59 seafile-data
```

Now we can start Seafile Server as user 'seafserver'
```sh
root@cloudserver:~# su -l seafserver
$ seafile-server-latest/seafile.sh start
$ seafile-server-latest/seahub.sh start
```

- [ admin email ] (enter the mail address you'll use as admin account)
- [ admin password ] (give it a password)
- [ admin password again ] (password again)

### Verification
Use nmap to check the necessary ports are open. 22 is SSH, only open if you installed SSH server.  3306 is mariadb, only bound to localhost, not accessible from outside via network. 8000 is seahub, the web interface. 8082 is seafile, the data service daemon:
```sh
$ nmap localhost

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-04 06:53 EDT
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000025s latency).
Other addresses for localhost (not scanned): ::1
Not shown: 996 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
3306/tcp open  mysql
8000/tcp open  http-alt
8082/tcp open  blackice-alerts

Nmap done: 1 IP address (1 host up) scanned in 0.03 seconds
$ nmap 192.168.1.2

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-04 06:59 EDT
Nmap scan report for 192.168.1.2
Host is up (0.000024s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
8000/tcp open  http-alt
8082/tcp open  blackice-alerts

Nmap done: 1 IP address (1 host up) scanned in 0.03 seconds
```
For a test open a web browser and log into your new Seafile Server:
```bash
http://192.168.1.2/8000/
```

Now stop Seafile Server
```sh
$ seafile-server-latest/seahub.sh stop
$ seafile-server-latest/seafile.sh stop
$ exit
```

There is some data located in the `/opt` directory. We should change this and move the data to `/srv`. A link will give Seafile Server access to its data:
```sh
root@cloudserver:~# mv /opt/Seafile/Server/seahub-data /srv/Seafile/
root@cloudserver:~# ln -s /opt/Seafile/Server/seahub-data /srv/Seafile/seahub-data
```

Append some lines to `/opt/Seafile/Server/conf/seahub_settings.py` to enable memcached.

```
...
        'HOST': '127.0.0.1',
        'PORT': '3306'
    }
}

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```
At least start your Seafile Server again as user 'seafserver' to check it's still working. Stop Seafile Server before proceeding to the next step.

## Systemd service files
For a convenient start of Seafile Server we need some appropriate definition files for the operating system. Debian 9 uses systemd as init system, so we create service files for systemd. Create a file ` /etc/systemd/system/seafile.service` with the following contents:
```
[Unit]
Description=Seafile
# add mysql.service or postgresql.service depending on your database to the line below
After=network.target mysql.service

[Service]
Type=oneshot
ExecStart=/opt/Seafile/Server/seafile-server-latest/seafile.sh start
ExecStop=/opt/Seafile/Server/seafile-server-latest/seafile.sh stop
RemainAfterExit=yes
User=seafserver
Group=seafserver

[Install]
WantedBy=multi-user.target
```

Create another file `/etc/systemd/system/seahub.service` with this contents:
```
[Unit]
Description=Seafile hub
After=network.target seafile.service

[Service]
# change start to start-fastcgi if you want to run fastcgi
ExecStart=/opt/Seafile/Server/seafile-server-latest/seahub.sh start
ExecStop=/opt/Seafile/Server/seafile-server-latest/seahub.sh stop
User=seafserver
Group=seafserver
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Reload the systemd configuration:
```sh
root@cloudserver:~# systemctl daemon-reload
```

Now you should be able to start your Seafile Server like any other ordinary service:
```sh
root@cloudserver:~# systemctl start seafile
root@cloudserver:~# systemctl start seahub
```

Verify if it is working (web browser: `http://192.168.1.2:8000/`)

You can stop Seafile Server:
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
```

To start Seafile Server at system startup the services need to be enabled:
```sh
root@cloudserver:~# systemctl enable seafile
Created symlink /etc/systemd/system/multi-user.target.wants/seafile.service → /etc/systemd/system/seafile.service.
root@cloudserver:~# systemctl enable seahub
Created symlink /etc/systemd/system/multi-user.target.wants/seahub.service → /etc/systemd/system/seahub.service.
```

To verify the automatic startup you need to reboot your server and afterwards Seafile Server should be running.

## Backup Installation
It is advised to have a Backup of the configuration. A little script will do this task. If you have the possibility to perform an easy backup of the whole server you can do it that way as well.
### Download the backup script
Download the script [BackupSeafileInstall](https://raw.githubusercontent.com/DerDanilo/Seafile-Best-Practise/master/BackupSeafileInstall "BackupSeafileInstall") and save it in `/usr/local/sbin/`.

Make it executable:
```sh
root@cloudserver:~# chmod 764 /usr/local/sbin/BackupSeafileInstall
```

### How it works
The script will save the configuration of Seafile Server, MariaDB (MySQL), Nginx and Let's Encrypt. The last two only if installed. Seafile Server installation is backed up completely, but no data.
The backups are tarballs (gzipped tar files). By default they go to `/srv/Backup/SeafileInstall/` and contain the creation date and time in their filenames. You can configure the backup location within the script.
Before performing the backup, the affected services will be stopped and started afterwards if they have been running before. This is needed because the database (MySQL od MariaDB) is saved on filesystem basis, which needs the service to be stopped!
### Parameters
Get it from the script itself:
```sh
root@cloudserver:~# BackupSeafileInstall --help
```

### Configuration
Edit the script and see what's in there to be configurable.
### Perform the backup
```sh
root@cloudserver:~# BackupSeafileInstall
```

### Test the result
See if it is there:
```sh
root@cloudserver:~# ls -l /srv/Backup/SeafileInstall
total 49740
-rw-r--r-- 1 root root 50932569 Jul 16 12:42 SeafileInstall201707161242.tgz
```

Verify its contents:
```sh
root@cloudserver:~# tar -tzf /srv/Backup/SeafileInstall/SeafileInstall201707161242.tgz | less
```

It should contain:
```
etc/systemd/system/seafile.service
etc/systemd/system/seahub.service
var/lib/mysql/
var/lib/mysql/seahub@002ddb/
var/lib/mysql/seahub@002ddb/base_userlastlogin.frm
...
var/lib/mysql/performance_schema/
var/lib/mysql/performance_schema/db.opt
opt/Seafile/Server/
opt/Seafile/Server/.bashrc
opt/Seafile/Server/seafile-server-latest
opt/Seafile/Server/seafile-server-6.1.1/
opt/Seafile/Server/seafile-server-6.1.1/setup-seafile.sh
...
opt/Seafile/Server/logs/seafile.log
opt/Seafile/Server/logs/seahub_django_request.log
opt/Seafile/Server/.bash_logout
opt/Seafile/Server/pids/
```

Of course if you have installed Nginx or Let's Encrypt later on, you should also see entries for that.
### Verify it's working
Just do a restore and verify it is working afterwards. If you want to be really sure not to lose something you might copy the current installation to a save location.
Stop the services. The last one of course only if installed.
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
root@cloudserver:~# systemctl stop mysql
root@cloudserver:~# systemctl stop nginx
```

Create a temporary directory to save the current installation and configuration:
```sh
root@cloudserver:~# mkdir -p /srv/SavedBeforeRestore/{etc,opt,var/log}
```

Move the current installation and configuration to the save area. Omit the parts which are not installed:
```sh
root@cloudserver:~# mv /etc/systemd/system/seafile.service /srv/SavedBeforeRestore/etc/
root@cloudserver:~# mv /etc/systemd/system/seahub.service /srv/SavedBeforeRestore/etc/
root@cloudserver:~# mv /var/lib/mysql /srv/SavedBeforeRestore/var/
root@cloudserver:~# mv /opt/Seafile/Server /srv/SavedBeforeRestore/opt/
root@cloudserver:~# mv /etc/nginx /srv/SavedBeforeRestore/etc/
root@cloudserver:~# mv /etc/letsencrypt /srv/SavedBeforeRestore/etc/
root@cloudserver:~# mv /var/log/nginx /srv/SavedBeforeRestore/var/log/
root@cloudserver:~# mv /var/log/letsencrypt /srv/SavedBeforeRestore/var/log/
```

Restore a backup:
```sh
root@cloudserver:~# tar -C / -xzf /srv/Backup/SeafileInstall/SeafileInstall201707161242.tgz
```

Reboot the server and see if everything ist working as before. If it does not, you can recover your last installation from the files in `/srv/SavedBeforeRestore/`. If it works well you can remove the saved files:
```sh
root@cloudserver:~# rm -rf /srv/SavedBeforeRestore
```

### Restore a backup
Stop the services. The last one of course only if installed.
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
root@cloudserver:~# systemctl stop mysql
root@cloudserver:~# systemctl stop nginx
```

Remove the current stuff you want to get rid of. Be sure to save anything you want to keep from the current installation like letsencrypt logfiles a.s.o.
```sh
root@cloudserver:~# rm -rf /etc/systemd/system/{seafile,seahub}.service /etc/nginx /etc/letsencrypt /opt/Seafile/Server /var/lib/mysql /var/log/letsencrypt /var/log/nginx
```

Restore a backup:
```sh
root@cloudserver:~# tar -C / -xzf /srv/Backup/SeafileInstall/SeafileInstall201707161242.tgz
```

Reboot the server.

## Backup Data
To have a Backup of the data in Seafile Server a little script will do this task.
### Download the backup script
Download the script [BackupSeafileData](https://raw.githubusercontent.com/DerDanilo/Seafile-Best-Practise/master/BackupSeafileData "BackupSeafileData") and save it in `/usr/local/sbin/`.

Make it executable:
```sh
root@cloudserver:~# chmod 764 /usr/local/sbin/BackupSeafileData
```

### How it works
The script dumps the Seafile database into a file in the Seafile Server data area `/srv/Seafile`. If requested the oldest backups will be deleted until a maximum number of backups is kept. This is configurable, the default is to keep everything and delete nothing. Then a tarball (gzipped tar file) will be created. The default location is `/srv/Backup/SeafileData/` and can be configured within the script. Finally the database dump will be removed. There is no need to stop any service, you can perform the backup on the fly.
### Parameters
Get it from the script itself:
```sh
root@cloudserver:~# BackupSeafileData --help
```

### Configuration
Edit the script and see what's in there to be configurable. You may set the database user and his password. If you don't do so, the script tries to get these values from the Seafile Server configuration.
```sh
MYSQL_USER=""
MYSQL_PASSWORD=""
```

The most interesting parameter might be:
```sh
KEEP_OLD=0
```

which disabled deletion of old backups completely. Change it to your needs, e.g. `KEEP_OLD=20` will keep 20 old backups (21 backups including the current) and removes all older ones. Keep in mind that you need plenty of space in the backup area to save o lot of backups. Watch out not to run out of disk space!
### Perform the backup
```sh
root@cloudserver:~# BackupSeafileData
```

### Test the result
See if it is there:
```sh
root@cloudserver:~# ls -l /srv/Backup/SeafileData
total 540
-rw-r--r-- 1 root root 550485 Jul 16 15:29 SeafileData201707161529.tgz
```

Verify its contents:
```sh
root@cloudserver:~# tar -tzf /srv/Backup/SeafileData/SeafileData201707161529.tgz | less
```

It should contain something like (important is `Seafile/seafile.sql`, the database backup):
```sh
Seafile/
Seafile/seahub-data/
Seafile/seahub-data/avatars/
Seafile/seahub-data/avatars/groups/
Seafile/seahub-data/avatars/groups/default.png
...
Seafile/seahub-data/avatars/default.png
Seafile/seafile-data/
Seafile/seafile-data/commits/
Seafile/seafile-data/library-template/
Seafile/seafile-data/library-template/seafile-tutorial.doc
Seafile/seafile-data/fs/
Seafile/seafile-data/storage/
...
Seafile/seafile-data/httptemp/
Seafile/seafile-data/tmpfiles/
Seafile/seafile.sql
```
### Verify it's working
Log via webbrowser into your Seafile Server. Modify something (upload a file, add a library or whatever).

Just do a restore of your backup and verify it is working afterwards and its state is reset to the time of the backup.

If you want to be really sure not to lose something you might copy the current data to a save location.

Stop the Seafile services, keep everything else running.
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
```

Move Seafile Data to a save location:
```sh
root@cloudserver:~# mv /srv/Seafile /srv/SavedBeforeRestore
```

Unpack the backup:
```sh
root@cloudserver:~# tar -C /srv -xzf /srv/Backup/SeafileData/SeafileData201707161529.tgz 
```

Verify the database backup contains reasonable data:
```sh
root@cloudserver:~# head /srv/Seafile/seafile.sql
-- MySQL dump 10.16  Distrib 10.1.23-MariaDB, for debian-linux-gnu (x86_64)
--
-- Host: localhost    Database: 
-- ------------------------------------------------------
-- Server version	10.1.23-MariaDB-9+deb9u1

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
```

Restore the database (assume 'seafile' is the name of your database user):
```sh
root@cloudserver:~# mysql -u seafile -p < /srv/Seafile/seafile.sql
Enter password:
```

Remove the database backup:
```sh
root@cloudserver:~# rm /srv/Seafile/seafile.sql
```

Start Seafile Server
```sh
root@cloudserver:~# systemctl start seafile
root@cloudserver:~# systemctl start seahub
```

Log via webbrowser into your Seafile Server and verify it's how it should be.

Remove the saved Data:
```sh
root@cloudserver:~# rm -rf /srv/SavedBeforeRestore
```

### Restore a backup
Stop the services, remove current data, restore a backup, restore database backup, remove database backup and start services again:
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
root@cloudserver:~# rm -rf /srv/Seafile
root@cloudserver:~# tar -C /srv -xzf /srv/Backup/SeafileData/SeafileData201707161529.tgz 
root@cloudserver:~# mysql -u seafile -p < /srv/Seafile/seafile.sql
Enter password: 
root@cloudserver:~# rm /srv/Seafile/seafile.sql
root@cloudserver:~# systemctl start seafile
root@cloudserver:~# systemctl start seahub
```

### Speed it up
If your server is able to run multiple threads in parallel, i.e. it has more than one core available or Intel's Hyper-Threading Technology you might be interested in speeding up your backup jobs. Normally tarballs are generated using gzip, which utilizes only one core. `pigz` is a fully functional replacement for gzip which utilizes multiple processors and multiple cores. If 'pigz' is installed, the backup scripts will use it to speed up the creation of backups.

Optionally you might want to install pigz:
```sh
root@cloudserver:~# apt-get install pigz
```

### Create a cronjob for a daily data backup
Edit crontab for user root and add a line at the end:
```sh
root@cloudserver:~# crontab -u root -e
# Edit this file to introduce tasks to be run by cron.
...
# m h  dom mon dow   command
0 3 * * * /usr/local/sbin/BackupSeafileData
```

This will create a Seafile data backup every day at 3:00 am. Adjust it to your needs.

**Watch your disk space!**

## IPv6
If you don't want to use or cannot use [IPv6](https://en.wikipedia.org/wiki/IPv6 "IPv6") you may skip this chapter and anything related to IPv6 in the following chapters.

### Test for IPv6 being enabled
```sh
root@cloudserver:~# ip -6 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2003:a:452:e300:5054:ff:feae:a412/64 scope global mngtmpaddr dynamic 
       valid_lft 7197sec preferred_lft 3597sec
    inet6 fe80::5054:ff:feae:a412/64 scope link 
       valid_lft forever preferred_lft forever
```

If you find an inet6 address with scope 'global' and not 'deprecated' and not 'temporary', it's an IPv6 address you may use to access your server.

If your server has such a global IPv6 address, you can test it with a web browser on a computer in local LAN having IPv6 enabled as well: `http://[2003:a:452:e300:5054:ff:feae:a412]`. This works because the Nginx 'default' server (which we have still enabled) comes with IPv6 enabled.

You can test it with curl as well:
```sh
root@cloudserver:~# curl -I http://[2003:a:452:e300:5054:ff:feae:a412]
HTTP/1.1 200 OK
Server: nginx/1.10.3
Date: Wed, 19 Jul 2017 07:31:05 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Mon, 17 Jul 2017 13:41:49 GMT
Connection: keep-alive
ETag: "596cbe9d-264"
Accept-Ranges: bytes
```

Or test it using nmap:
```sh
root@cloudserver:~# nmap -6 2003:a:452:e300:5054:ff:feae:a412

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-19 09:33 CEST
Nmap scan report for 2003:a:452:e300:5054:ff:feae:a412
Host is up (0.000015s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Note that port 80 is open because of the default server, port 443 is not!

### Enabling IPv6 for Seafile Server

Add a line behind the 'listen 443' directive in `/etc/nginx/sites-available/seafile` like this:
```
...
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  _;
...
```

We will refrain from adding it to port 80 as well because of `server_name  192.168.1.2;`. Of course we could add a second name and... No, not now!

Restart Nginx:
```sh
root@cloudserver:~# systemctl restart nginx
```

Test it using nmap:
```sh
root@cloudserver:~# nmap -6 fe80::5054:ff:feae:a412

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-19 10:13 CEST
Nmap scan report for fe80::5054:ff:feae:a412
Host is up (0.000015s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

Open a web browser: `https://[2003:a:452:e300:5054:ff:feae:a412]/seafile`. Mind the 'https:' as there is no redirection for now. If you see the Log In page, it works. Do not log in for the moment because in Seafile Server we set our IPv4 address as SERVICE_URL and FILE_SERVER_ROOT. It will mix up things sooner or later if you log in. Just be satisfied with the knowledge it could work if we would continue. But in that case we would break IPv4.

## Seafile Server with Nginx
Nginx is a web server, which we will use as a reverse proxy. In this mode incoming requests can distributed to several services, in our case to the seafile and seahub services. Furthermore Nginx can secure the connection to the browsers or clients providing encryption through TLS protocol a.s.o.
### Install Nginx
```sh
root@cloudserver:~# apt-get install nginx
```

Verify it is running. Open a web browser: `http://192.168.1.2`. It should show the default page "Welcome to nginx!".

### Principles of nginx configuration
Read the [Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html "Beginner’s Guide"), which gives an idia of the block structure used in the configuration. You can have several server blocks, each defining a different port or server name. The server blocks itself may contain any number of location blocks, which define the action needed to handle the request at that specific location. An action can be a forwarding to another server or to a service. This is reverse proxying, which we will use for seafile and seahub daemons. Another action is simply delivering a static object. We will use this for delivering icons, avatars, JavaScript and the like.
### Configuring the server for using Nginx
One of the downsides of our configuration is not having configured our DNS properly. We gave our server a name, but it is not reachable via network using that name. So we are not able to access it like `http://cloudserver`.

Nginx comes with a default configuration, defining a server listening on port 80 (default for http) named '\_', which is a catch-all that is taken if no other name/port combination matches. We don't want to switch that off, so we need an acceptable server name because we do not want to use a different port. We will use our IP address for now as the name of our virtual Nginx server.

### Create a basic Nginx configuration for Seafile Server
Create a file `/etc/nginx/sites-available/seafile` with the following contents (adjust the IP adress in 'server_name'):
```
server {
    listen       80;
    server_name  192.168.1.2;
    server_tokens off;

    location /seafile {
        fastcgi_pass    127.0.0.1:8000;
        fastcgi_param   SCRIPT_FILENAME     $document_root$fastcgi_script_name;
        fastcgi_param   PATH_INFO           $fastcgi_script_name;

        fastcgi_param   SERVER_PROTOCOL     $server_protocol;
        fastcgi_param   QUERY_STRING        $query_string;
        fastcgi_param   REQUEST_METHOD      $request_method;
        fastcgi_param   CONTENT_TYPE        $content_type;
        fastcgi_param   CONTENT_LENGTH      $content_length;
        fastcgi_param   SERVER_ADDR         $server_addr;
        fastcgi_param   SERVER_PORT         $server_port;
        fastcgi_param   SERVER_NAME         $server_name;
        fastcgi_param   REMOTE_ADDR         $remote_addr;
        fastcgi_param   HTTPS               off;
        fastcgi_param   HTTP_SCHEME         http;

        fastcgi_read_timeout                36000;

        access_log      /var/log/nginx/seahub.access.log;
        error_log       /var/log/nginx/seahub.error.log;
    }

    location /seafhttp {
        rewrite ^/seafhttp(.*)$ $1 break;
        proxy_pass http://127.0.0.1:8082;
        client_max_body_size                0;
        proxy_connect_timeout               36000s;
        proxy_read_timeout                  36000s;
        proxy_send_timeout                  36000s;
        send_timeout                        36000s;
        proxy_request_buffering             off;
    }

    location /seafmedia {
        rewrite ^/seafmedia(.*)$ /media$1 break;
        root /opt/Seafile/Server/seafile-server-latest/seahub;
    }

    location /seafdav {
        fastcgi_pass    127.0.0.1:8080;
        fastcgi_param   SCRIPT_FILENAME     $document_root$fastcgi_script_name;
        fastcgi_param   PATH_INFO           $fastcgi_script_name;

        fastcgi_param   SERVER_PROTOCOL     $server_protocol;
        fastcgi_param   QUERY_STRING        $query_string;
        fastcgi_param   REQUEST_METHOD      $request_method;
        fastcgi_param   CONTENT_TYPE        $content_type;
        fastcgi_param   CONTENT_LENGTH      $content_length;
        fastcgi_param   SERVER_ADDR         $server_addr;
        fastcgi_param   SERVER_PORT         $server_port;
        fastcgi_param   SERVER_NAME         $server_name;

        fastcgi_param   HTTPS               off;
        proxy_request_buffering             off;

        client_max_body_size 0;

        access_log      /var/log/nginx/seafdav.access.log;
        error_log       /var/log/nginx/seafdav.error.log;
    }
}
```
The most interesting parts of this configuration are:
```
listen 80;               The port Nginx listens to
server_name 192.168.1.2; The 'name' of the virtual server
server_tokens off;       Nginx does not reveal its version number to make life more difficult for attackers
location /seafile        proxy for seahub (!)
location /seafhttp       proxy for seafile (!)
location /seafmedia      static content of Seafile Server
location /seafdav        a goodie for you, we don't use it for now
access_log and error_log Nginx log files
```

Stop Nginx:
```sh
root@cloudserver:~# systemctl stop nginx
```

Enable the seafile configuration in Nginx:
```sh
root@cloudserver:~# ln -s /etc/nginx/sites-available/seafile /etc/nginx/sites-enabled/seafile
```

### Adjusting Seafile Server for use with Nginx
Stop Seafile Server:
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
```

Adjust 'SERVICE_URL' in `/opt/Seafile/Server/conf/ccnet.conf` (mind the IP address):
```
SERVICE_URL = http://192.168.1.2/seafile
```

Add some lines in `/opt/Seafile/Server/conf/seahub_settings.py` between `SECRET_KEY` and `DATABASES` (mind the IP address):
```
SECRET_KEY = ...

FILE_SERVER_ROOT = 'http://192.168.1.2/seafhttp'

SERVE_STATIC = False
MEDIA_URL = '/seafmedia/'
SITE_ROOT = '/seafile/'
LOGIN_URL = '/seafile/accounts/login/'
COMPRESS_URL = MEDIA_URL
STATIC_URL = MEDIA_URL + 'assets/'

DATABASES = ...
```

Adjust `ExecStart` in `/etc/systemd/system/seahub.service` to tell seahub it is used in conjunction with Nginx:
```
ExecStart=/opt/Seafile/Server/seafile-server-latest/seahub.sh start-fastcgi
```

At least a little security enhancement. We will bind seafile service to localhost only which makes it reachable through Nginx only. Add a line in `/opt/Seafile/Server/conf/seafile.conf` between `[fileserver]` and `port = 8082` like:
```
[fileserver]
host = 127.0.0.1
port = 8082
...
```

Reload systemd configuration and start the whole thing:
```sh
root@cloudserver:~# systemctl daemon-reload
root@cloudserver:~# systemctl start seafile
root@cloudserver:~# systemctl start seahub
root@cloudserver:~# systemctl start nginx
```

If there are no errors reported, you can test it with your web browser: `http://192.168.1.2/seafile`.

If everything looks fine, it is time for a backup of the installation!

### Troubleshooting
If you get an error when starting a service it is a good hint in which parts of the configuration the problem has its cause. We modified ccnet.conf which affects seafile and seahub. seafile.conf affects seafile. seahub.service and seahub_settings.py affect seahub. The Nginx configuration affects only Nginx!

Check the ports. They need to be open for localhost.
- 22: openssh (only open if installed)
- 80: nginx
- 3306: mysql (or mariadb)
- 8000: seahub
- 8082: seafile

With the exeption of port 22 the only port open from LAN should be port 80.
```sh
root@cloudserver:~# nmap localhost

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-18 11:23 CEST
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000090s latency).
Other addresses for localhost (not scanned): ::1
Not shown: 995 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
8000/tcp open  http-alt
8082/tcp open  blackice-alerts

Nmap done: 1 IP address (1 host up) scanned in 1.64 seconds
root@cloudserver:~# nmap 192.168.1.2

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-18 11:27 CEST
Nmap scan report for 192.168.1.2
Host is up (0.000010s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.64 seconds
```

If all ports are open as required, you may do some more tests:

### Testing Nginx configuration
You can test the Nginx configuration for syntax errors:
```sh
root@cloudserver:~# nginx -t -c /etc/nginx/nginx.conf
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Nginx default server:
```sh
root@cloudserver:~# curl -I http://192.168.1.2
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 18 Jul 2017 10:19:27 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 31 Jan 2017 15:01:11 GMT
Connection: keep-alive
ETag: "5890a6b7-264"
Accept-Ranges: bytes
```

Nginx seafile file delivery:
```sh
root@cloudserver:~# curl -I http://192.168.1.2/seafmedia/img/seafile-logo.png
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 18 Jul 2017 09:48:47 GMT
Content-Type: image/png
Content-Length: 12612
Last-Modified: Tue, 13 Jun 2017 05:49:44 GMT
Connection: keep-alive
ETag: "593f7cf8-3144"
Accept-Ranges: bytes
```

If you get 'HTTP/1.1 404 Not Found', did you adjust the IP address in `/etc/nginx/sites-available/seafile`? Have you set the link in `/etc/nginx/sites-enabled/` to enable it? 'Server: nginx/1.10.3' is an indication, that the 'default server' is taken, not the 'seafile server'. And yes, we should turn that off, but at this point it's helpful. 'Server: nginx': Paths in `location /seafmedia` in `/etc/nginx/sites-available/seafile` and the Seafile Server installation do not match. Is the file present:
```sh
root@cloudserver:~# ls -l /opt/Seafile/Server/seafile-server-latest/seahub/media/img/seafile-logo.png
-rw-rw-r-- 1 seafserver seafserver 12612 Jun 13 07:49 /opt/Seafile/Server/seafile-server-latest/seahub/media/img/seafile-logo.png
```

'HTTP/1.1 403 Forbidden': Check the filesystem access rights 
for `/opt/Seafile/Server/seafile-server-latest/seahub/media/img/seafile-logo.png`. Each directory in the whole path must have set read and execute bit for others (drwxr-x**r**-**x**), the file itself must be world readable (-rwxr-x**r**--).

Avatar Icon damaged: check filesystem rights for `/srv/Seafile/seahub-data/avatars/default.png`. The whole path must be world readable.

Nginx and seahub daemon:
```sh
root@cloudserver:~# curl -I http://192.168.1.2/seafile/
HTTP/1.1 302 FOUND
Server: nginx
Date: Tue, 18 Jul 2017 11:00:27 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Vary: Accept-Language, Cookie
Location: http://192.168.1.2/seafile/accounts/login/?next=/seafile/
Content-Language: en
```

If you dont get the 'HTTP/1.1 302 FOUND', there is a problem between seahub daemon and Nginx. Activated `start-fastcgi` in `seahub.service`?

Still problems? Look into the log-files in `/var/log/nginx/`, `/opt/Seafile/Server/logs/` and `/var/log/messages`.

If you got it working, back up the installation.

## Access from Internet

Depending on your internet connection your server might be accessible from internet via IPv4, IPv6, both or not at all.
### root server or vServer
You are done, skip to next chapter.
### IPv6 Behind NAT router in local LAN
Set up port forwarding in your router:
- port 80/TCP to IP 192.168.1.2 port 80
- port 443/TCP to IP 192.168.1.2 port 443

Get your public IPv4 address:
```sh
root@cloudserver:~# wget -qO- http://ipecho.net/plain; echo
62.224.170.64
```

Do a test from outside your LAN (smartphone with a mobile connection will be sufficiant) using a web browser: `http://62.224.170.64/seafile`. If you can see your Seafile Server, do not log in, it will break things! If it does not work, you are probably behind a Carrier-grade NAT, Dual Stack Lite or whatever prevents a connection from internet to your server. Sorry, but no IPv4 access to your Seafile Server.

### IPv6 behind router in local LAN
Enable IPv6 access from internet to your server for port 80 and port 443 in your router.

Do a test from outside your LAN using a web browser:  `https://[2003:a:452:e300:5054:ff:feae:a412]/seafile`. A smartphone using a mobile connection will do, if it provides IPv6 internet access. Be sure to have IPv6 internet access. You can test it using a service like `http://test-ipv6.com/`.

If you can reach your Seafile Server, don't log in! If you cannot see your Seafile Server it is probably behind a firewall of your internet service provider. Sorry, but no IPv6 access to your Seafile Server.

## Domain Name
A Domain Name is essential to use a trusted X.509 certificate. If you want the web browsers to accept your certificate, you need a domain name.
### Static or dynamic IP / prefix?
If you don't know, if you have a static IP, you probably have a dynamic IP. That means your public IP changes from time to time. It may change after each login of your router to your internet service provider, daily, after some months or whenever. For IPv4 you normally get one IP for your router, which can forward requests to your LAN. For IPv6 you get a whole network, from which IPv6 addresses are distributed to the devices in your network. The common part of these addresses is called prefix.

### Static IP / prefix
You can either register a domain like 'seafile.com' (that one is already assigned to sombody else, choose another one) and point it to your IPv4 and / or IPv6 address or create a subdomain in it like 'home.seafile.com' and do the same with the subdomain. Or you can ignore that it's static and do the same as it was a dynamic IP.

### Dynamic IP / prefix
If you have a dynamic IP, you need a [DDNS](https://en.wikipedia.org/wiki/Dynamic_DNS "DDNS") provider.

You can register a domain and do what your provider recommends to update the IP address. If you choose one for free, you will get a name like 'subdomainspecialforyou.bigddnsprovider.tld'. There is nothing wrong with it, but if you want to register an X.509 certificate for it, make sure it will be possible with the certificate authority of your choice. Let's Encrypt for example has [Rate Limits](https://letsencrypt.org/docs/rate-limits/ "Rate Limits") which prevent users of 'bigddnsprovider.tld' to register too many certificates within a time interval. A hosting provider may request a higher rate limit for Let's Encrypt, but if your favorite provider didn't do that, you will run into problems getting or renewing a certificate. Check that before you choose a DDNS provider and want to use Let's Encrypt. This is no problem if you register your own domain.
### Dynamic IPv4
Because all devices in LAN share the public IPv4 address, each of the devices may do the update of the DDNS name. Let the router it, if is able to. That's normally the most stable solution. Your server should be the next choice.
### Dynamic IPv6
Because each device gets its own IPv6 address, it is up to your server to do the update.
### Implement DDNS update, configure DNS
If it's static DNS, configure it to point to your IP address(es). DDNS users should test the update mechanism to make sure, it allways points to your current IP address(es). Switch your router off and on and whatever you can think of to ensure, it's working.
### Configure Seafile Server for your domain
If you don't have a public IPv4 address, don't configure an IPv4 address for your domain! If you don't have a public IPv6 address, don't configure an IPv6 address for your domain! You may configure your public IPv4 address and your global IPv6 address of your server, if you can reach your server with both protocols.

Configure `/etc/nginx/sites-available/seafile` to be web server for your domain. IPv6 users may also enable port 80 for IPv6. Adjust the server name to your needs:
```
server {
    listen       80;
    listen       [::]:80;
    server_name  home.seafile.com;
    server_tokens off;
...

server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  home.seafile.com;
    server_tokens off;
...
```

Configure `/opt/Seafile/Server/conf/ccnet.conf` for your domain:
```
...
SERVICE_URL = https://home.seafile.com/seafile
...
```

Configure `/opt/Seafile/Server/conf/seahub_settings.py` for your domain:
```
...
FILE_SERVER_ROOT = 'https://home.seafile.com/seafhttp'
...
```

Restart services:
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl restart seafile
root@cloudserver:~# systemctl start seahub
root@cloudserver:~# systemctl restart nginx
```

Test it with your web browser: `http://home.seafile.com/seafile`. If it does not work, try from outside your LAN. If that works, but from inside your LAN it does not, it's probably a problem with [hairpinning](https://en.wikipedia.org/wiki/Hairpinning "hairpinning") (also known as NAT loopback). In most cases, the router simply does not support it. You could try [split DNS](https://en.wikipedia.org/wiki/Split-horizon_DNS "split DNS"), but that's evil and I didn't tell you. Better try a different router.
### Troubleshooting
If you can log into your Seafile Server but uploading or viewing files failes, check your `SERVICE_URL` and `FILE_SERVER_ROOT` in seafile settings again.

Do not forget your backups!

## Seafile Server with Nginx HTTPS
First of all, switch off gzip compression if using https. It's a security risk:
```sh
root@cloudserver:~# sed -i 's/\tgzip/\t# gzip/' /etc/nginx/nginx.conf
```

For TLS (HTTPS) we need a [digital certificate](https://en.wikipedia.org/wiki/Public_key_certificate "digital certificate"), which proves the ownership of a [public key](https://en.wikipedia.org/wiki/Public_key "public key"). That means, we need to create a pair of keys (public and private part), and then sign them by ourselves. We can use this self-signed certificate for TLS, but it will not be accepted by the browsers. Must browsers allow to enter an exeption for our server so we can live with it for the moment.

Modern cryptography offers [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy "forward secrecy"), a protection against future compromises, which requires [Diffie–Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange "Diffie–Hellman key exchange"). We need to generate the parameters for it. The Certificate Signing Request will ask you some things required for the certificate. Just fill in what you think it's right. It will be visible in the certificate.
```sh
root@cloudserver:~# cd /etc/ssl/private
root@cloudserver:/etc/ssl/private# openssl genrsa -out privkey.pem 2048
root@cloudserver:/etc/ssl/private# openssl req -new -x509 -key privkey.pem -out cacert.pem -days 3650
root@cloudserver:/etc/ssl/private# openssl dhparam -outform PEM -out dhparam2048.pem 2048
root@cloudserver:/etc/ssl/private# cd
```

Verify the result:
```sh
root@cloudserver:~# ls -l /etc/ssl/private
total 12
-rw-r--r-- 1 root root 1379 Jul 18 15:46 cacert.pem
-rw-r--r-- 1 root root  424 Jul 18 15:51 dhparam2048.pem
-rw------- 1 root root 1679 Jul 18 15:44 privkey.pem
```

Modify `/etc/nginx/sites-available/seafile` to be:
```
server {
    listen       80;
    server_name  192.168.1.2;
    server_tokens off;

    location /seafile {
      rewrite ^ https://$http_host$request_uri? permanent;    # force redirect http to https
    }
}

server {
    listen       443 ssl http2;
    server_name  _;
    server_tokens off;

    ssl_protocols TLSv1.2;
    ssl_certificate /etc/ssl/private/cacert.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;
    ssl_dhparam /etc/ssl/private/dhparam2048.pem;
    ssl_ecdh_curve secp384r1;
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM:EECDH:EDH:!MD5:!RC4:!LOW:!MEDIUM:!CAMELLIA:!ECDSA:!DES:!DSS:!3DES:!NULL;
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 10m;

    location /seafile {
...
        fastcgi_param   HTTPS               on;
        fastcgi_param   HTTP_SCHEME         https;
...
    location /seafdav {
...
        fastcgi_param   HTTPS               on;
...
```

The most interesting parts are:
```
rewrite ^ https://$http_host$request_uri? permanent; # Redirect http to https
listen  443 ssl http2;    listen to port 443, enable TLS and accept HTTP/2
ssl_protocols TLSv1.2;    accept TLS 1.2 only
```

Restart Nginx
```sh
root@cloudserver:~# systemctl restart nginx
```

If there are no errors reported, you can test it with your web browser: `http://192.168.1.2/seafile`. If everything works, your browser should redirect to https and tell you something like 'Your connection is not secure'. You should be able to add an exception to access your Seafile Server.

If everything looks fine you could back up the installation.

### Troubleshooting
We mainly modified the Nginx configuration, so there is a big chance for the problem to be in there.

Check the ports are open:
```sh
root@cloudserver:~# nmap 192.168.1.2

Starting Nmap 7.40 ( https://nmap.org ) at 2017-07-18 16:28 CEST
Nmap scan report for 192.168.1.2
Host is up (0.000010s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

Check the http part, it should be a redirection (Moved Permanently) to https:
```sh
root@cloudserver:~# curl -I http://192.168.1.2/seafile/
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Tue, 18 Jul 2017 14:48:43 GMT
Content-Type: text/html
Content-Length: 178
Connection: keep-alive
Location: https://192.168.1.2/seafile/
```

Check the https part (option '-k' for curl disables verification of the certificate):
```sh
root@cloudserver:~# curl -I -k https://192.168.1.2/seafile/
HTTP/2 302 
server: nginx
date: Tue, 18 Jul 2017 14:52:16 GMT
content-type: text/html; charset=utf-8
location: https://192.168.1.2/seafile/accounts/login/?next=/seafile/
vary: Accept-Language, Cookie
content-language: en
```

If you got it working, back up the installation.

## Digital Certificate
At this point your server should be reachable by your domain, but our self-signed certificate is not trusted.

We will get an X.509 certificate from [Let's Encrypt](https://letsencrypt.org/ "Let's Encrypt") for free, which will be accepted by almost every actual web browser.

Let's Encrypt uses the ACME protocol to ensure you control the domain. We need a client to get a certificate and to renew it before it expires.
### Install Certbot and configure the system
We will use [Certbot](https://certbot.eff.org/ "Certbot"), the standard ACME client from Let's Encrypt. Install it:
```sh
root@cloudserver:~# apt-get install certbot
```
Certbot provides several methods to obtain certificates, They are called plugins. We will use the plugin 'webroot', which will use an already running webserver to deliver some content to the Let's Encrypt CA (certification authority) to prove we are controlling the domain. The benefit of this plugin is no need for us to stop Nginx when renewing a certificate, a simple reload of Nginx' configuration is sufficient to deliver the new certificate. Debian comes with a nice feature to automate the renewing of certificates. Unfortunately it is buggy because it does not reload the changes in configuration into Nginx. The second point we don't like, it runs as root. That's enough to turn it off and write our own solution:
```sh
root@cloudserver:~# systemctl disable certbot.timer
```

Create a user to run certbot:
```sh
useradd -U -m certbot
```

`/etc/letsencrypt` will contain the Let's Encrypt certificate(s), `/var/log/letsencrypt` the log files for certbot and `var/www/letsencrypt` we will use for Certbot and Nginx to handle the domain validation. The first two are created by the installation, we only need to set the access rights (Nginx runs as www-data:www-data)
```sh
root@cloudserver:~# mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
root@cloudserver:~# chown certbot:certbot /etc/letsencrypt /var/log/letsencrypt
root@cloudserver:~# chmod 750 /var/www/letsencrypt
root@cloudserver:~# chown -R certbot:www-data /var/www/letsencrypt
```

Now create a location block in `/etc/nginx/sites-available/seafile` in the http server (port 80) behind the `location /seafile` block to support the ACME domain validation:
```
...
    location /seafile {
      rewrite ^ https://$http_host$request_uri? permanent;    # force redirect http to https
    }

    location /.well-known/acme-challenge {
        alias /var/www/letsencrypt/.well-known/acme-challenge;
        location ~ /.well-known/acme-challenge/(.*) {
            add_header Content-Type application/jose+json;
        }
    }
}

server {
...
```

Restart Nginx:
```sh
root@cloudserver:~# systemctl restart nginx
```

### OCSP Stapling and Must-Staple
At this point we are ready to obtain our certificate. Before we'll do that, we have to make a decision. Generally, if a private key ist thought to have been compromised, the certificate should be revoked. Let's Encrypt will publish that revocation information through [OCSP](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol "OCSP"), the Online Certificate Status Protocol. The basic idea is, a browser should check the revocation status via OCSP. The problem is, the OCSP servers are under heavy load and it is not guaranteed the web browser will get an answer within a reasonable time. So the web browsers ignore the OCSP answer if it takes too long, or don't contact the OSCP server at all.

An extension to the TLS protocol, called [OCSP stapling](https://en.wikipedia.org/wiki/OCSP_stapling "OCSP stapling"), solves this problem. If used, the web server (our Nginx) gets a ticket from the OSCP server and delivers it along with the certificate. So the web browser gets an assurance from the CA that the certificate is not revoked.

The problem with this is, the client does not know if the web server will send an OCSP response. If an attacker tries to abuse a stolen, revoked certificate he can just block the connection to the OCSP server an deliver the certificate without OCSP stapling. The web browser will just accept the connection.

The solution is [OCSP Must-Staple](https://casecurity.org/2014/06/18/ocsp-must-staple/ "OCSP Must-Staple"). We can have a flag set in the certificate, which tells the web browsers we will deliver an OCSP response for sure. If we don't do so, the browser shall hard-fail, which means it should't accept the connection.

The decision you have to make is wheather you want the OCSP Must-Staple indication in your certificate or not.

### Obtain the certificate
Remove the '--must-staple' from command line if you don't want the OCSP Must-Staple flag in your certificate. Replace the 'home.seafile.com' to your domain name. Obtain the certificate as user 'certbot'. Answer the questions.
```sh
root@cloudserver:~# su -l certbot
$ certbot certonly --webroot -w /var/www/letsencrypt -d home.seafile.com
$ exit
```

If it tells you 'Congratulations! Your certificate and chain have been saved at ...' everything worked and you have obtained your certificate. If not, the error message should point you into the right direction.

### Configure Nginx to use the certificate
Edit `/etc/nginx/sites-available/seafile` to point to your Let's Encrypt certificate. We'll leave the old self-signed certificate in the configuration, but comment it out:
```
...
    ssl_protocols TLSv1.2;
#    ssl_certificate /etc/ssl/private/cacert.pem;
#    ssl_certificate_key /etc/ssl/private/privkey.pem;
    ssl_certificate /etc/letsencrypt/live/home.seafile.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/home.seafile.com/privkey.pem;
    ssl_dhparam /etc/ssl/private/dhparam2048.pem;
...
```

If you want OCSP Stapling (or need it because your certificate requires it), edit the configuration again to enable OCSP Stapling. 'resolver' is any DNS resolver, reachable by your server. You may use that one in your router or whatever you like:
```
...
    ssl_certificate /etc/letsencrypt/live/home.seafile.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/home.seafile.com/privkey.pem;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 192.168.1.1;
    ssl_dhparam /etc/ssl/private/dhparam2048.pem;
...
```

Restart Nginx:
```sh
root@cloudserver:~# systemctl restart nginx
```

If you requested OCSP Must-Staple in your certificate, and you test it with your web browser `http://home.seafile.com/seafile`, it might fail. This is because OSCP Stapling in Nginx is broken. Nginx sends out the first TLS connection without stapling even it's enabled. If your test didn't give you an error, the OCSP handling of your browser is also broken. If you load your web site a second time, it will work because Nginx then already has obtained the OCSP response and delivers it.

### Workaround for the broken OCSP Stapling in Nginx
The handling of OCSP Stapling of Nginx is at least bad, and in case of Must-Staple, almost unusable.

If you are using OSCP Stapling, create the script `/usr/local/sbin/OCSPResponse` with the following contents. if you like, you may set the SERVER variable to your domain:
```
#!/bin/bash
# OCSPResponse
# Get OSCP respone for certificate
VERSION="20170723"

# you might set your domain name
SERVER=""

# try to get SERVER from configuration if not set
[ "$SERVER" = "" ] && SERVER="$(ls /etc/letsencrypt/live/)"

# Location of the certificate
CHAIN="/etc/letsencrypt/live/$SERVER/chain.pem"
CERT="/etc/letsencrypt/live/$SERVER/cert.pem"

# Location where to store a valid OCSP response
OCSPRESPONSE=/etc/nginx/OCSP/$SERVER/ocsp.response

# some temporary locations
TMPREPLY=/tmp/ocsp.reply
TMPRESPONSE=/etc/nginx/OCSP/ocsp.response

# Create the required paths
mkdir -p ${OCSPRESPONSE%/*}

# Get the OCSP server url out of the certificate
OCSPURL="$(openssl x509 -noout -ocsp_uri -in $CERT)"

# get the OCSP response and save it
openssl ocsp -no_nonce -respout $TMPRESPONSE -issuer $CHAIN -verify_other $CHAIN -cert $CERT -url $OCSPURL >$TMPREPLY 2>/dev/null

# Check the reply for being valid (drop response, if request failed)
if [ "$(grep ': good' $TMPREPLY)" != "" ]; then
  # move response to its final location
  mv $TMPRESPONSE $OCSPRESPONSE
  # reload nginx
  systemctl reload nginx
fi

# Remofe temporary stuff
rm -f $TMPREPLY $TMPRESPONSE
```

Make the script executable, run it and verify the result:
```sh
root@cloudserver:~# chmod 750 /usr/local/sbin/OCSPResponse
root@cloudserver:~# OCSPResponse
root@cloudserver:~# ls -l /etc/nginx/OCSP/home.seafile.com/ocsp.response 
-rw-r--r-- 1 root root 527 Jul 23 13:02 /etc/nginx/OCSP/home.seafile.com/ocsp.response
```

Edit `/etc/nginx/sites-available/seafile` to use the response file:
```
...
   ssl_stapling on;
   ssl_stapling_file /etc/nginx/OCSP/home.seafile.com/ocsp.response;
#   ssl_stapling_verify on;
#   resolver 192.168.1.1;
    ssl_dhparam /etc/ssl/private/dhparam2048.pem;
...
```

Restart Nginx and verify it's working with your web browser.

Create a cronjob to get a new OCSP reponse once a day. Change the time to a different value to prevent a daily DDOS if millions of Seafile Servers update their OCSP response at the same time at seven minutes past five :-)

```sh
root@cloudserver:~# crontab -u root -e
# Edit this file to introduce tasks to be run by cron.
...
# m h  dom mon dow   command
0 3 * * * /usr/local/sbin/BackupSeafileData
7 5 * * * /usr/local/sbin/OCSPResponse
```

### Certificate renewal
Because we disabled Debian's certificate renewal, we have to create our own. Create a file `/usr/local/sbin/CertbotRenew` with the following contents:
```
#!/bin/bash
# CertbotRenew
# Renew Let's Encrypt certificate
VERSION="20170723"

# Renew certificate as user certbot
su certbot -c "certbot renew"
 
# Get a new OCSP response if configured
if [ -x /usr/local/sbin/OCSPResponse ]; then
    /usr/local/sbin/OCSPResponse # will reload nginx
else
  systemctl reload nginx
fi
```

Make it executable and create a cronjob to execute it once a week. Be nice to Let's Encrypt and change day and time:
```sh
root@cloudserver:~# chmod 754 /usr/local/sbin/CertbotRenew 
root@cloudserver:~# crontab -u root -e
# Edit this file to introduce tasks to be run by cron.
...
# m h  dom mon dow   command
0 3 * * * /usr/local/sbin/BackupSeafileData
7 5 * * * /usr/local/sbin/OCSPResponse
9 7 * * 3 /usr/local/sbin/CertbotRenew
```

Backup your Installation.

