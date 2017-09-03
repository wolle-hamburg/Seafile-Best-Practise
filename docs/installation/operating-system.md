[TOC]

---

## Install Operating system
In this manual we will use Debian 9 *"Stretch"* 64-bit as operating system. It can be obtained from [https://www.debian.org/](https://www.debian.org/). The "64-bit PC Network installer" will be sufficiant.
In this manual we will put the seafile installation and configuration in `/opt` and seafile_data in `/srv`. If you devide your disk(s) into several partitions, 10 GiB will be enough for `/`. If you have up to 2 GiB of memory, choose 2 GiB for swap, otherwise 1 GiB will be sufficiant. `/srv` will contain all data in seafile server and needs to be at least that size plus some spare for deleted files.