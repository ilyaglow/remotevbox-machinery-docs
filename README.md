About
-----

This guide will cover a case of Virtualbox web service machinery usage that shows how a Cuckoo can work inside a VirtualBox VM itself achieving a nested analysis feature. There could other setups, including [dockerized one](https://github.com/blacktop/docker-cuckoo/tree/master/vbox) to remotely control Virtualbox machines, without giving an access to the whole hypervisor - the only thing you need is a shared storage between Virtualbox Webservice daemon and Cuckoo (see below).

# How it works

Traditional Virtualbox machinery works over a command line `vboxmanage` application, which has obvious limitations - Cuckoo Sandbox should work on the same host that manages analysis virtual machines. To bypass this limitation [remotevbox](https://github.com/ilyaglow/remote-virtualbox) library was written. It leverages [Virtualbox Webservice](https://www.virtualbox.org/manual/ch09.html#vboxwebsrv-daemon) feature by talking to it via SOAP API.

# Prerequisites

* Host - Linux-based distro with Virtualbox installed.
* Prepared (with vmcloak for example) virtual machine to perform an analysis.
* Virtual machine with Cuckoo installed. You can find a lot of guides how to do it right. If you are familiar with Docker and feel adventurous you can try a dockerized version from here https://github.com/ilyaglow/docker-cuckoo/tree/vbox-work - a modified version of @blacktop's [docker-cuckoo project](https://github.com/blacktop/docker-cuckoo).

# Host
## User management

It is recommended to use a non-root user that will run Virtualbox Webservice. Couple reasons:
* When I started to research it and figuring out how to use it, everything seemed ancient - SDK-library, documentation. The C++ code of the web service is at least 3 years old.
* Virtualbox Webservice is a SOAP service, which means it parses XML using C++ (hello binary vulns!).
* It uses `gSOAP` toolkit which was involved in [Devil's Ivy vulnerability](https://blog.senr.io/blog/devils-ivy-flaw-in-widely-used-third-party-code-impacts-millions) not long time ago.


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

Attach network adapters of both virtual machines to `vboxnet0` host-only interface. Change IP-addresses of the Cuckoo and analysis VMs.

For the further simplicity we'll make it like the following:
* Analysis VM `192.168.56.2`
* Cuckoo VM `192.168.56.100`

If you want analysis VM to be able to connect to external world, you should do proper routing, but keep in mind that the Cuckoo Rooter obviously won't work as you might expect, because Cuckoo is not running on our host.

So you probably want just a `Simple Global Routing`: https://docs.cuckoosandbox.org/en/latest/installation/guest/network/

## Virtualbox Webservice

Ensure that a file `/etc/default/virtualbox` has the following format:

```
VBOXWEB_HOST=192.168.56.1 # IP reachable from Cuckoo VM
VBOXWEB_USER=cuckoo       # created user
```

Actually you can disable authentication at all for debugging purposes and even for security *reasons* - no credentials to steal from the config files and no credentials to sniff over the wire (but of course you can easily use reverse proxy for vboxwebservice terminate TLS on it, its just a SOAP service after all):
```bash
cuckoo@host:$ vboxmanage setproperty websrvauthlibrary null
```

## Shared storage

The key requirement for the Virtualbox web service machinery to work is a shared storage that is used to store analysis dumps by Virtualbox Webservice daemon and Cuckoo VM, so be cautious about permissions. Assume you have `cuckoo` user on the Cuckoo VM with a strong password and openssh server installed. Your `$CUCKOO_CWD` here is `/home/cuckoo/.cuckoo`:

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

## Virtualbox web service machinery

Install remotevbox library on Cuckoo VM:
```
cuckoo@cuckoovm:$ pip install -U remotevbox --user
```

Save the [virtualbox_webserv.conf](https://raw.githubusercontent.com/blacktop/docker-cuckoo/blob/master/vbox/conf/virtualbox_websrv.conf) to your `$CUCKOO_CWD/conf` directory and change the file to reflect your current settings:

* Change `url` to `https://192.168.56.1:18083`
* Set `user` and `password` to credentials of a cuckoo user on a host. You can empty its values, if you disabled authentication earlier.
* Set `remote_storage` to your shared storage on a **host**, in our case it is `/mnt/share`.
* Add your prepared analysis machine to `machines` section.
* Set `ip` of the machine to IP-address in your host-only network, in our case `192.168.56.2`.
* `resultserver_ip` according to previous step should be `192.168.56.100`.

### Starting the daemon

There are two options:
* Start as a service with `systemctl`:
```
root@host:# systemctl start vboxweb-service
```

* Start as a binary `vboxwebsrv` in a background mode:
```
cuckoo@host:$ vboxwebsrv --background --host 192.168.56.1
```

**IMPORTANT**: Due to undefined yet reasons on some Virtualbox package builds the service do not respect `/etc/default/virtualbox` and start itself as root. It is recommended to check this behaviour by using a simple `ps aux | grep -i vboxweb` after the service is started.


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

Now you can browse `http://192.168.56.100:8080` from your host.

# Network hardening
## Restrict access to host interface
* to vboxwebservice and host IP 192.168.56.1 only from a Cuckoo VM:
```bash
root@host:# iptables -A INPUT -i vboxnet0 -m conntrack --ctstate ESTABLISHED,RELATED -s 192.168.56.100 -p tcp --destination-port 18083 --destination 192.168.56.1 -j ACCEPT
root@host:# iptables -A INPUT -i vboxnet0 --destination 192.168.56.1 -j DROP
```

## Restrict access to Cuckoo VM
* to Cuckoo resultserver from `192.168.56.0/24` subnet:
```bash
root@cuckoovm:# iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -s 192.168.56.0/24 -p tcp --destination-port 2042 -j ACCEPT
```

* to ssh and Cuckoo web from `192.168.56.1` only
```bash
root@cuckoovm:# iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -s 192.168.56.1 -p tcp --destination-port 22 -j ACCEPT
root@cuckoovm:# iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -s 192.168.56.1 -p tcp --destination-port 8080 -j ACCEPT
root@cuckoovm:# iptables -A INPUT -j DROP
```
