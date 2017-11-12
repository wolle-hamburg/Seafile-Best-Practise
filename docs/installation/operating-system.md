[TOC]

---
## General information

This manual covers setup on the following system:

* Debian (latest)
* Ubuntu LTS (latest)

To be written:

* CentOS (latest)
* Raspbian (Raspberry Pi)
* Docker (Linux, maybe Windows)

### PATH (folder) information

* In this manual we will put the Seafile installation and configuration in `/opt` and seafile_data in `/srv`
* `/srv` will contain all data in Seafile server and needs to be at least that data size plus some spare for deleted files (file versions)

* If you devide your disk(s) into several partitions or want to use several disks, 10 GiB will be enough for `/` (root)
* If you have 2+ GiB of memory, choose 2 GiB for swap, otherwise 1 GiB will be sufficiant

## Debian
You can use Debian 9 *"Stretch"* 64-bit as operating system.  
The *"64-bit PC Network installer"* will be sufficiant. It can be obtained from [www.debian.org](https://www.debian.org/distrib/). 

## Ubuntu LTS
You can use Ubuntu 16.04 LTS *"Xenial"* 64-bit as operating system.  
The *"64-bit PC Network installer"* will be sufficiant. It can be obtained from [www.ubuntu.com](https://www.ubuntu.com/download/alternative-downloads). 

## CentOS
Not yet written.

## Raspbian (Raspberry Pi)
You can use  Debian 9 *"Raspbian"* as operating system.   
We recommend to use the *"Minimal image"*. It can be obtained from [(www.raspberrypi.org](https://www.raspberrypi.org/downloads/raspbian/).

## Docker (Linux & Windows)
You may also use the [offical docker image](https://hub.docker.com/r/seafileltd/seafile/) of Seafile server.
There are also some 3rd Party Seafile Docker images that might work better.

*The manual does not yet cover this topic.*