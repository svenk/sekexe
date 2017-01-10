# SEparate KErnel eXEcution

SEKEXE uses User Mode Linux to run a Linux process within a "sub kernel",
and retrieve its output and exit status (as if it was running as a normal
shell command). This is half-way between full-fleged virtual machines
and lightweight containers: VMs can run any O/S; containers run the same
O/S and kernel as their host; SEKEXE only runs Linux-inside-Linux, but the
guest can be a different version.

This is useful if you want to:

- isolate a process, but your kernel doesn't support containers or other
  isolation features;
- run or test specific kernel features that are not supported by your
  current kernel;
- run a process as root, but do not have root privileges.
- have special needs for networking.

SEKEXE was originally developed for running docker as an unpriviledged user.
You find the repository at [jpetazzo/sekexe](https://github.com/jpetazzo/sekexe).

I use SEKEXE mainly for passing firewalls in multi-user systems (i.e. the HPC
world).

## Networking

User Mode Linux allows (non-root) networking directly with
[slirp]https://en.wikipedia.org/wiki/Slirp) or with
[vde](http://vde.sourceforge.net/). VDE allows to set up virtual switches and
plugs which evventually may end at a TAP interface at a remote linux computer.
Thus, you can have the full IP, including seamless ping/ICMP/DNS/DHCP,
without having any root privileges on the host computer.

For a typical setup, I run

```
user@lockedhost $ vde_switch -s /tmp/switch1 -M /tmp/mgt1 &
user@lockedhost $ dpipe vde_plug /tmp/switch1 = nc remote.example.com 1234 &
```

and at the other computer I either directly connect to a slirp router

```
user@lockedhost $ dpipe nc -klp 1234 = slirpvde -s - --dhcp
```

or use the tap interface (root needed). It requires a one-time setup like

```
root@outhost # sudo tunctl -u uml-net
root@outhost # sudo ip link set dev tap0 up
root@outhost # sudo brctl addif virbr0 tap0
root@outhost # sudo brctl addif virbr0 eth1
```

to bridge `tap0` to `eth1` and delegate the permissions from `root` to 
a user `uml-net` which then can run the switch and connect a listening daemon:

```
uml-net@outhost $ vde_switch -s ./sockets/tap0switch -M ./sockets/mgt_tap0switch --tap tap0 &
uml-net@outhost $ dpipe nc -klp 1234 = vde_plug sockets/tap0switch &
```

This setup allows to pipe all networking performed by the SEKEXE instance
runned by `user@lockedhost` throught a single TCP or UDP host (also ssh
tunneling is an option) which is suitable to go throught some
restrictive firewalls.

## Further information

This repository contains a recent UML binary which is based on Linux 4.10 and distinguishes
from the original [jpetazzo/sekexe](https://github.com/jpetazzo/sekexe) kernel by supporting
all kind of filesystems for mounting loopback images. You can read off the `.config` by calling
`./uml4 --showconfig`.

Read the original Readme file containing more information about how to use
it at [jpetazzo/sekexe](https://github.com/jpetazzo/sekexe).
