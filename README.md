This guide will cover a case of remotevbox machinery usage that shows how Cuckoo management services can work inside VirtualBox VM achieving a nested analysis feature.

# How it works

Traditional Virtualbox machinery works over a command line `vboxmanage` application, which has obvious limitations - Cuckoo Sandbox should works on the same host that manages analysis virtual machines. To bypass this limitation [remotevbox](https://github.com/ilyaglow/remote-virtualbox) library was written. It leverages [Virtualbox Webservice](https://www.virtualbox.org/manual/ch09.html#vboxwebsrv-daemon) feature by talking to it via SOAP API.

# Prerequisites

* Host - Linux-based distro with Virtualbox installed
* Prepared (with vmcloak for example) virtual machine to perform an analysis
* Virtual machine with Cuckoo installed. You can find a lot of guides how to do it right. If you are familiar with docker and feel adventerous you can try a dockerized version from here https://github.com/ilyaglow/docker-cuckoo/tree/vbox-work - a modified version of @blacktop's [docker-cuckoo project](https://github.com/blacktop/docker-cuckoo).

# Configure host
## User management

It is recommended to use a non-root user that will run Virtualbox Webservice. Couple reasons:
* When I started to research it and figuring out how to use it, everything seemed ancient - sdk library, documentation. The C++ code of it is at least 3 years old.
* Virtualbox Webservice is a SOAP service, which means it parses XML using C++ (hello binary vulns!).
* It uses gSOAP toolkit which was involved with [Devil's Ivy vulnerability](https://blog.senr.io/blog/devils-ivy-flaw-in-widely-used-third-party-code-impacts-millions) not long time ago.


```bash
adduser cuckoo
```

Keep in mind that you should import virtual machines for analysis by this user.

## Network

Create a host-only network:
```bash
cuckoo@cuckoohost:$ vboxmanage hostonlyif create
```

Set host IP-address:
```bash
cuckoo@cuckoohost:$ vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
```

Attach network adapters of both virtual machines to vboxnet0 host-only interface. And change IP-address of the analysis and cuckoo vms.

For the further simplicity we'll make it like the following:
* Analysis VM `192.168.56.2`
* Cuckoo VM `192.168.56.100`

## Configure Virtualbox Webservice

Ensure that a file `/etc/default/virtualbox` has a following format:

```
VBOXWEB_HOST=192.168.56.1 # IP reachable from cuckoo vm
VBOXWEB_USER=cuckoo       # created user
```

Actually you can disable auth at all and it has its own valid *reasons* - no credentials to read from config files, no credentials to sniff over the wire (but of course you can easily reverse proxy your vboxwebservice behind nginx and terminate TLS on it):
```bash
cuckoo@cuckoohost:$ vboxmanage setproperty websrvauthlibrary null
```

## Configure a shared storage

The key requirement for remotevbox machinery to works is a shared storage that is used to store analysis dumps by Virtualbox Webservice and Cuckoo VM, so be cautious about permissions.

```bash
cuckoo@cuckoohost:$ sshfs vbox@192.168.56.100:/mnt/share /home/cuckoo/share
```

# Cuckoo VM configuring

## Configuring machinery

# Network hardening
