[TOC]

---

## Update MUST READ

* Don't forget to stop Seafile Server before the upgrade
e.g.: `service seafile-server stop` or `service seahub stop && service seafile stop` or or `systemctl stop seahub seafile`
* Don't forget to apply the upgrade script/s according your start release with the properly user which in normal case should be `seafile` 
e.g. change to seafile user: `su seafile` or `sudo su seafile`
* Don't forget to change the directory rights of Seafile after unpacking it
e.g.: `chown -R seafile:nogroup ./seafile-server-6.2.1` or `sudo chown -R seafile:seafile ./seafile-server-6.2.1`


## Updating

### Download latest stable server version

### Stop seafile services
`service seafile-server stop` or `service seahub stop && service seafile stop` or or `systemctl stop seahub seafile`

### Untar seafile server package

### Apply update


#### Minor update

#### Major update


### Start services
`service seafile-server start` or `service seahub start && service seafile start` or or `systemctl start seafile && systemctl start seahub`

### Verify update

### Move old server package

`mv ./seafile-server-6.2.1 ./oldversions` and `mv ./seafile-server_6.2.1.tar.gz ./oldversions`