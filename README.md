About
-----

This guide will cover a case of remotevbox machinery usage that shows how Cuckoo management services can work inside VirtualBox VM achieving a nested analysis feature. Of course you can build your own network setup and remotely control Virtualbox machines - the only thing you need is a shared storage between Virtualbox Web Service daemon and Cuckoo (see below).

# How it works

Traditional Virtualbox machinery works over a command line `vboxmanage` application, which has obvious limitations - Cuckoo Sandbox should works on the same host that manages analysis virtual machines. To bypass this limitation [remotevbox](https://github.com/ilyaglow/remote-virtualbox) library was written. It leverages [Virtualbox Webservice](https://www.virtualbox.org/manual/ch09.html#vboxwebsrv-daemon) feature by talking to it via SOAP API.

# Prerequisites

* Host - Linux-based distro with Virtualbox installed
* Prepared (with vmcloak for example) virtual machine to perform an analysis
* Virtual machine with Cuckoo installed. You can find a lot of guides how to do it right. If you are familiar with docker and feel adventerous you can try a dockerized version from here https://github.com/ilyaglow/docker-cuckoo/tree/vbox-work - a modified version of @blacktop's [docker-cuckoo project](https://github.com/blacktop/docker-cuckoo).

# Host
## User management

It is recommended to use a non-root user that will run Virtualbox Webservice. Couple reasons:
* When I started to research it and figuring out how to use it, everything seemed ancient - sdk library, documentation. The C++ code of it is at least 3 years old.
* Virtualbox Webservice is a SOAP service, which means it parses XML using C++ (hello binary vulns!).
* It uses gSOAP toolkit which was involved with [Devil's Ivy vulnerability](https://blog.senr.io/blog/devils-ivy-flaw-in-widely-used-third-party-code-impacts-millions) not long time ago.


```bash
root@host:# adduser cuckoo
```

Keep in mind that you should import virtual machines for analysis by this user.

## Network

Create a host-only network:
```bash
cuckoo@host:$ vboxmanage hostonlyif create
```

Set host IP-address:
```bash
cuckoo@host:$ vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
```

Attach network adapters of both virtual machines to vboxnet0 host-only interface. And change IP-address of the analysis and cuckoo vms.

For the further simplicity we'll make it like the following:
* Analysis VM `192.168.56.2`
* Cuckoo VM `192.168.56.100`

## Virtualbox Webservice

Ensure that a file `/etc/default/virtualbox` has a following format:

```
VBOXWEB_HOST=192.168.56.1 # IP reachable from cuckoo vm
VBOXWEB_USER=cuckoo       # created user
```

Actually you can disable auth at all and it has its own valid *reasons* - no credentials to steal from config files, no credentials to sniff over the wire (but of course you can easily reverse proxy your vboxwebservice behind nginx and terminate TLS on it):
```bash
cuckoo@host:$ vboxmanage setproperty websrvauthlibrary null
```

## Shared storage

The key requirement for remotevbox machinery to work is a shared storage that is used to store analysis dumps by Virtualbox Webservice and Cuckoo VM, so be cautious about permissions. Assume you have `cuckoo` user on Cuckoo VM with strong password and openssh server installed. You `$CUCKOO_CWD` here is `/home/cuckoo/.cuckoo`:

```bash
root@host:# mkdir /mnt/share && chown cuckoo:cuckoo /mnt/share
root@host:# su -l cuckoo
cuckoo@host:$ sshfs cuckoo@192.168.56.100:/mnt/share /home/cuckoo/.cuckoo/storage
```

Now you have a shared storage in `/mnt/share` on a host that maps to `/home/cuckoo/.cuckoo/storage` on a Cuckoo VM.

Check that the `cuckoo` user has permissions to write to this storage:
```bash
cuckoo@host:$ echo > /mnt/share/test_file
```

# Cuckoo VM

If you did a regular Cuckoo installation you should add `remotevbox-machinery` files by yourself - at the time of writing it isn't merged in upstream of Cuckoo.

The `/usr/lib/python2.7/site-packages/cuckoo` is the path to your Cuckoo installation that could be different: if you use `virtual-env` (you do not need to run `wget` as *root* in this case):

```
root@cuckoovm:# wget -O /usr/lib/python2.7/site-packages/cuckoo/machinery/virtualbox_websrv.py https://raw.githubusercontent.com/ilyaglow/cuckoo/blob/remotevbox-machinery/cuckoo/machinery/virtualbox_websrv.py
root@cuckoovm:# wget -O /usr/lib/python2.7/site-packages/cuckoo/common/config.py https://raw.githubusercontent.com/ilyaglow/cuckoo/blob/remotevbox-machinery/cuckoo/common/config.py
```

## Remotevbox machinery

Save the [virtualbox_webserv.conf](https://raw.githubusercontent.com/blacktop/docker-cuckoo/blob/master/vbox/conf/virtualbox_websrv.conf) to your `$CUCKOO_CWD/conf` directory and change the file to reflect your current settings:

* Change `url` to `https://192.168.56.1:18083`
* Set `user` and `password` to credentials of a cuckoo user on a host. You can empty its values, if you disabled authentication earlier.
* Set `remote_storage` to your shared storage on a **host**, in our case it is `/mnt/share`.
* Add your prepared analysis machine to `machines` section.
* Set `ip` of the machine to IP-address in your host-only network, in our case `192.168.56.2`.
* `resultserver_ip` according to previous step should be `192.168.56.100`.



## Cuckoo start

Drop privileges to cuckoo user and check permissions of the shared storage:
```bash
cuckoo@cuckoovm:$ echo > /home/cuckoo/.cuckoo/storage/testfile
```

Start cuckoo daemons (its useful to run them on screen or tmux):
```
cuckoo@cuckoovm:$ cuckoo -d
cuckoo@cuckoovm:$ cuckoo web runserver 192.168.56.100:8080
```

Now you can try to browse `http://192.168.56.100:8080` from your host.

# Network hardening

## Restrict access to host interface
* to vboxwebservice only from Cuckoo VM

## Restrict access to Cuckoo VM
* to resultserver from `192.168.56.0/24` subnet
* to ssh and Cuckoo web from `192.168.56.1` only
