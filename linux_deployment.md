# HOW TO: Linux best practise Seafile Server deployment
This manual will show step by how to setup a Seafile Server using MariaDB, Memcached and nginx as a reverse proxy on Debian operating system.

All operations will be peformed as root unless otherwise specified. So login as root or 'su' to root, if logged in as ordinary user:
```sh
su
```
## Install Operating system
In this manual we will use Debian 9 *"Stretch"* 64-bit as operating system. It can be obtained from [https://www.debian.org/](https://www.debian.org/). The "64-bit PC Network installer" will be sufficiant.
In this manual we will put the seafile installation and configuration in `/opt` and seafile_data in `/srv`. If you devide your disk(s) into several partitions, 10 GiB will be enough for `/`. Choose 2 GiB for swap if you have up to 2 GiB of RAM, otherwise 1 GiB will be sufficiant. `/srv` will contain all data in seafile server and needs to be at least that size plus some spare for deleted files.
## Static-IPv4
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

Now it is better to have a backup of the configuration. We can put it in our home directory (of the user 'root'):
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
```sh
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

[Debian Reference Manual: The network interface with the static IP](https://www.debian.org/doc/manuals/debian-reference/ch05.en.html#_the_network_interface_with_the_static_ip)
## Diagnostic Tools
Now is a good point to install some usefull tools for diagnostics you might need later on.

Install curl
```sh
root@cloudserver:~# apt-get install curl
```
curl is a tool to transfer data from or to a server. We will use it to diagnose the web server.

Install nmap
```sh
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
### Download and install Seafile Server
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
```sh
http://192.168.1.2/8000/
```
Now stop Seafile Server
```sh
$ seafile-server-latest/seahub.sh stop
$ seafile-server-latest/seafile.sh stop
$ exit
```
### Final changes
There is some data located in the `/opt` directory. We should change this and move the data to `/srv`. A link will give Seafile Server access to its data:
```sh
root@cloudserver:~# mv /opt/Seafile/Server/seahub-data /srv/Seafile/
root@cloudserver:~# ln -s /opt/Seafile/Server/seahub-data /srv/Seafile/seahub-data
```
Append some lines to 'seahub_settings.py' to enable memcached. You can use your preferred editor if you don't like the here document:

```sh
root@cloudserver:~# cat >> /opt/Seafile/Server/conf/seahub_settings.py << EOF
> 
> CACHES = {
>     'default': {
>         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
>         'LOCATION': '127.0.0.1:11211',
>     }
> }
> EOF
```
At least start your Seafile Server again as user 'seafserver' to check it's still working. Stop it before proceeding to the next step.
## Systemd service files
For a convenient start of Seafile Server we need some appropriate definition files for the operating system. Debian 9 uses systemd as init system, so we create service files for systemd:
```sh
root@cloudserver:~# cat > /etc/systemd/system/seafile.service << EOF
> [Unit]
> Description=Seafile
> # add mysql.service or postgresql.service depending on your database to the line below
> After=network.target mysql.service
> 
> [Service]
> Type=oneshot
> ExecStart=/opt/Seafile/Server/seafile-server-latest/seafile.sh start
> ExecStop=/opt/Seafile/Server/seafile-server-latest/seafile.sh stop
> RemainAfterExit=yes
> User=seafserver
> Group=seafserver
> 
> [Install]
> WantedBy=multi-user.target
> EOF
root@cloudserver:~# cat > /etc/systemd/system/seahub.service << EOF
> [Unit]
> Description=Seafile hub
> After=network.target seafile.service
> 
> [Service]
> # change start to start-fastcgi if you want to run fastcgi
> ExecStart=/opt/Seafile/Server/seafile-server-latest/seahub.sh start
> ExecStop=/opt/Seafile/Server/seafile-server-latest/seahub.sh stop
> User=seafserver
> Group=seafserver
> Type=oneshot
> RemainAfterExit=yes
> 
> [Install]
> WantedBy=multi-user.target
> EOF
```
Reload the systemd configuration:
```sh
root@cloudserver:~# systemctl daemon-reload
```
### Starting Seafile Server
Now you should be able to start your Seafile Server like any other ordinary service:
```sh
root@cloudserver:~# systemctl start seafile
root@cloudserver:~# systemctl start seahub
```
### Verification
Verify if it is working (web browser: `http://192.168.1.2:8000/`)
### Stopping Seafile Server
You can stop Seafile Server:
```sh
root@cloudserver:~# systemctl stop seahub
root@cloudserver:~# systemctl stop seafile
```
### Starting Seafile Server at system start
To start Seafile Server at system startup, the services need to be enabled:
```sh
root@cloudserver:~# systemctl enable seafile
Created symlink /etc/systemd/system/multi-user.target.wants/seafile.service → /etc/systemd/system/seafile.service.
root@cloudserver:~# systemctl enable seahub
Created symlink /etc/systemd/system/multi-user.target.wants/seahub.service → /etc/systemd/system/seahub.service.
```
To verify the automatic startup you need to reboot your server and afterwards Seafile Server should be running.
### Troubleshooting
You can verify Seafile Server is running (key 'q' quits the pager and puts you back to command shell):
```sh
root@cloudserver:~# systemctl status seahub
root@cloudserver:~# systemctl status seafile
```
Log files are in `/opt/Seafile/Server/logs` for Seafile Server, system log is in `/var/log/messages`. You can view the contents e.g. with a pager ('G' puts you to end of file, 'q' quits):
```sh
root@cloudserver:~# less /var/log/messages
```
## Backup the Installation
## Backup Seafile Data
## Install nginx and configure it as reverse proxy
## Switch to HTTPS using a self-signed certificate
