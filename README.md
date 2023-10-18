# Planète Bracca - Karim MEHADJBIA
# Séance 2

## Lancement machine virtuelle
```
karimmehadjbia@eve:~/seance2$ ./ovs-startup.sh machine1.qcow2 2048 111
~> Virtual machine filename   : machine1.qcow2
~> RAM size                   : 2048MB
~> SPICE VDI port number      : 6011
~> telnet console port number : 2411
~> MAC address                : b8:ad:ca:fe:00:6f
~> Switch port interface      : tap111, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:6f%vlan52
karimmehadjbia@eve:~/seance2$ telnet localhost 2411
Trying ::1...
Connected to localhost.
Escape character is '^]'.
```
## Configuration IP initiale
```
etu@machine1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:00:6f brd ff:ff:ff:ff:ff:ff
    inet 198.18.53.111/23 brd 198.18.53.255 scope global dynamic enp0s1
       valid_lft 86296sec preferred_lft 86296sec
    inet6 2001:678:3fc:34:baad:caff:fefe:6f/64 scope global dynamic mngtmpaddr proto kernel_ra
       valid_lft 86297sec preferred_lft 14297sec
    inet6 fe80::baad:caff:fefe:6f/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```

## Installation OvS

```
etu@machine1:~$ sudo apt -y install openvswitch-switch
[sudo] Mot de passe de etu :
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait
openvswitch-switch est déjà la version la plus récente (3.2.0-2).
0 mis à jour, 0 nouvellement installés, 0 à enlever et 15 non mis à jour.
etu@machine1:~$ apt search ^openvswitch-switch$
En train de trier... Fait
Recherche en texte intégral... Fait
openvswitch-switch/testing,now 3.2.0-2 amd64  [installé]
  Open vSwitch switch implementations
```

## Configuration interfaces

```
etu@machine1:~$ sudo cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s1
iface enp0s1 inet dhcp

face enp0s1 inet6 static
        address 2001:678:3fc:34:baad:caff:fefe:6f/64
        gateway fe80::6cc1:e3ff:fee7:df9e
            dns-nameservers 2001:678:3fc:3::2

auto C-3PO
iface C-3PO inet manual
        ovs_type OVSBridge
        ovs_ports sw-vlan10
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down

allow-C-3PO sw-vlan10
iface sw-vlan10 inet static
        ovs_type OVSBridge
        ovs_bridge C-3PO
        ovs_options C-3PO 10
        address 192.0.2.1/24

iface sw-vlan10 inet6 static
        ovs_type OVSBridge
        ovs_bridge C-3PO
        ovs_options C-3PO 10
        address fdc0:2::1/64
```
## Activation Routage

```

etu@machine1:~$ egrep -v '(^#|^$)' /etc/sysctl.conf
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
net.ipv4.conf.all.log_martians = 1
etu@machine1:~$ sudo sysctl --system
* Applique /usr/lib/sysctl.d/50-pid-max.conf …
* Applique /usr/lib/sysctl.d/99-protect-links.conf …
* Applique /etc/sysctl.d/99-sysctl.conf …
* Applique /etc/sysctl.conf …
kernel.pid_max = 4194304
fs.protected_fifos = 1
fs.protected_hardlinks = 1
fs.protected_regular = 2
fs.protected_symlinks = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.log_martians = 1
```

## Configuration dnsmasq

```
etu@machine1:~$ sudo apt -y install dnsmasq
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait
dnsmasq est déjà la version la plus récente (2.89-1).
0 mis à jour, 0 nouvellement installés, 0 à enlever et 15 non mis à jour.
etu@machine1:~$ apt search ^dnsmasq$
En train de trier... Fait
Recherche en texte intégral... Fait
dnsmasq/testing,now 2.89-1 all  [installé]
  petit mandataire cache DNS et serveur DHCP/TFTP
etu@machine1:~$ egrep -v '(^#|^$)' /etc/dnsmasq.conf
dnssec
server=9.9.9.9@enp0s1
server=2620:fe::fe@enp0s1
no-dhcp-interface=enp0s1
no-hosts
expand-hosts
domain=local.cloud
dhcp-range=192.0.2.10,192.0.2.100,1h
dhcp-range=fdc0:2::, ra-names
enable-ra
dhcp-option=option6:dns-server,[fdc0:2::1],[2620:fe::fe]
etu@machine1:~$ sudo systemctl restart dnsmasq.service
etu@machine1:~$
```

## iptables

```
etu@machine1:~$ sudo iptables -t nat -vnL
Chain PREROUTING (policy ACCEPT 2 packets, 656 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 10 packets, 695 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 1 packets, 67 bytes)
 pkts bytes target     prot opt in     out     source               destination
    9   628 MASQUERADE  0    --  *      enp0s1  0.0.0.0/0            0.0.0.0/0
etu@machine1:~$ sudo ip6tables -t nat -vnL
Chain PREROUTING (policy ACCEPT 2 packets, 160 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 1 packets, 80 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    80 MASQUERADE  0    --  *      enp0s1  ::/0                 ::/0
```

## Installation lxd

```
etu@machine1:~$ sudo apt -y install snapd
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait
snapd est déjà la version la plus récente (2.60.2-1+b1).
0 mis à jour, 0 nouvellement installés, 0 à enlever et 15 non mis à jour.
etu@machine1:~$ sudo snap install lxd
snap "lxd" is already installed, see 'snap help refresh'
etu@machine1:~$ sudo adduser etu lxd
info: Ajout de l'utilisateur « etu » au groupe « lxd » ...
etu@machine1:~$ id | grep -o lxd
lxd
```

## Initiation des containeurs


```
etu@machine1:~$ lxd init
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Name of the storage backend to use (btrfs, ceph, dir, lvm) [default=btrfs]:
Create a new BTRFS pool? (yes/no) [default=yes]:
Would you like to use an existing empty block device (e.g. a disk or partition)? (yes/no) [default=no]:
Size in GiB of the new loop device (1GiB minimum) [default=11GiB]:
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: yes
Name of the existing bridge or host interface: sw-vlan10
Would you like the LXD server to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
etu@machine1:~$ lxc profile device get default eth0 nictype
bridged
```

## Création des containeurs


```
etu@machine1:~$ for i in {0..2}; do lxc launch images:debian/12 c$i; done
Creating c0
Starting c0
Creating c1
Starting c1
Creating c2
Starting c2
etu@machine1:~$ lxc ls
+------+---------+-------------------+-----------------------------------+-----------+-----------+
| NAME |  STATE  |       IPV4        |               IPV6                |   TYPE    | SNAPSHOTS |
+------+---------+-------------------+-----------------------------------+-----------+-----------+
| c0   | RUNNING | 192.0.2.50 (eth0) | fdc0:2::216:3eff:fe42:da1a (eth0) | CONTAINER | 0         |
+------+---------+-------------------+-----------------------------------+-----------+-----------+
| c1   | RUNNING | 192.0.2.65 (eth0) | fdc0:2::216:3eff:fe53:9731 (eth0) | CONTAINER | 0         |
+------+---------+-------------------+-----------------------------------+-----------+-----------+
| c2   | RUNNING | 192.0.2.74 (eth0) | fdc0:2::216:3eff:fe72:2b8d (eth0) | CONTAINER | 0         |
+------+---------+-------------------+-----------------------------------+-----------+-----------+
```

## Test du premier conteneur

```
etu@machine1:~$ lxc exec c0 -- /bin/bash
root@c0:~# ip addr ls dev eth0
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:42:da:1a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.0.2.50/24 metric 1024 brd 192.0.2.255 scope global dynamic eth0
       valid_lft 3390sec preferred_lft 3390sec
    inet6 fdc0:2::216:3eff:fe42:da1a/64 scope global mngtmpaddr noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fe42:da1a/64 scope link
       valid_lft forever preferred_lft forever
root@c0:~# apt update
Hit:1 http://deb.debian.org/debian bookworm InRelease
Get:2 http://deb.debian.org/debian bookworm-updates InRelease [52.1 kB]
Get:3 http://deb.debian.org/debian-security bookworm-security InRelease [48.0 kB]
Get:4 http://deb.debian.org/debian-security bookworm-security/main amd64 Packages [79.5 kB]
Get:5 http://deb.debian.org/debian-security bookworm-security/main Translation-en [45.9 kB]
Fetched 225 kB in 1s (218 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
root@c0:~# for i in {1..2}; do ping -c2 c$i; done
PING c1(c1.local.cloud (fdc0:2::216:3eff:fe53:9731)) 56 data bytes
64 bytes from c1.local.cloud (fdc0:2::216:3eff:fe53:9731): icmp_seq=1 ttl=64 time=1.06 ms
64 bytes from c1.local.cloud (fdc0:2::216:3eff:fe53:9731): icmp_seq=2 ttl=64 time=0.114 ms

--- c1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.114/0.588/1.063/0.474 ms
PING c2(c2.local.cloud (fdc0:2::216:3eff:fe72:2b8d)) 56 data bytes
64 bytes from c2.local.cloud (fdc0:2::216:3eff:fe72:2b8d): icmp_seq=1 ttl=64 time=0.939 ms
64 bytes from c2.local.cloud (fdc0:2::216:3eff:fe72:2b8d): icmp_seq=2 ttl=64 time=0.071 ms

--- c2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.071/0.505/0.939/0.434 ms


```

## Automatisation premiers pas

```
etu@machine1:~$ for i in {0..2}; do lxc exec c$i -- apt update; done
Hit:1 http://deb.debian.org/debian bookworm InRelease
Hit:2 http://deb.debian.org/debian bookworm-updates InRelease
Get:3 http://deb.debian.org/debian-security bookworm-security InRelease [48.0 kB]
Fetched 48.0 kB in 1s (62.2 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
Hit:1 http://deb.debian.org/debian bookworm InRelease
Get:2 http://deb.debian.org/debian bookworm-updates InRelease [52.1 kB]
Get:3 http://deb.debian.org/debian-security bookworm-security InRelease [48.0 kB]
Get:4 http://deb.debian.org/debian-security bookworm-security/main amd64 Packages [79.5 kB]
Get:5 http://deb.debian.org/debian-security bookworm-security/main Translation-en [45.9 kB]
Fetched 225 kB in 1s (273 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
Hit:1 http://deb.debian.org/debian bookworm InRelease
Get:2 http://deb.debian.org/debian bookworm-updates InRelease [52.1 kB]
Get:3 http://deb.debian.org/debian-security bookworm-security InRelease [48.0 kB]
Get:4 http://deb.debian.org/debian-security bookworm-security/main amd64 Packages [79.5 kB]
Get:5 http://deb.debian.org/debian-security bookworm-security/main Translation-en [45.9 kB]
Fetched 225 kB in 1s (275 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.

```


## Verification ports OvS et TCAM

```
etu@machine1:~$ sudo ovs-vsctl show
[sudo] Mot de passe de etu :
4136ad74-2e4c-46fe-8739-8705f3240576
    Bridge C-3PO
        Port veth39dffec9
            tag: 10
            Interface veth39dffec9
        Port sw-vlan10
            tag: 10
            Interface sw-vlan10
                type: internal
        Port C-3PO
            Interface C-3PO
                type: internal
        Port vethcf4248db
            tag: 10
            Interface vethcf4248db
        Port vethc203dc36
            tag: 10
            Interface vethc203dc36
    ovs_version: "3.2.0"
etu@machine1:~$ ping -c2 ff02::1%sw-vlan10
PING ff02::1%sw-vlan10(ff02::1%sw-vlan10) 56 data bytes
64 bytes from fe80::d05a:97ff:fe08:24a%sw-vlan10: icmp_seq=1 ttl=64 time=0.136 ms
64 bytes from fe80::216:3eff:fe42:da1a%sw-vlan10: icmp_seq=1 ttl=64 time=1.02 ms
64 bytes from fe80::216:3eff:fe53:9731%sw-vlan10: icmp_seq=1 ttl=64 time=1.07 ms
64 bytes from fe80::216:3eff:fe72:2b8d%sw-vlan10: icmp_seq=1 ttl=64 time=1.09 ms
64 bytes from fe80::d05a:97ff:fe08:24a%sw-vlan10: icmp_seq=2 ttl=64 time=0.072 ms

--- ff02::1%sw-vlan10 ping statistics ---
2 packets transmitted, 2 received, +3 duplicates, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.072/0.677/1.090/0.469 ms
etu@machine1:~$ ip nei ls dev sw-vlan10
192.0.2.74 lladdr 00:16:3e:72:2b:8d STALE
192.0.2.50 lladdr 00:16:3e:42:da:1a STALE
192.0.2.65 lladdr 00:16:3e:53:97:31 STALE
fdc0:2::216:3eff:fe72:2b8d lladdr 00:16:3e:72:2b:8d STALE
fe80::216:3eff:fe72:2b8d lladdr 00:16:3e:72:2b:8d STALE
fdc0:2::216:3eff:fe42:da1a lladdr 00:16:3e:42:da:1a STALE
fe80::d05a:97ff:fe08:24a lladdr d2:5a:97:08:02:4a router STALE
fe80::216:3eff:fe42:da1a lladdr 00:16:3e:42:da:1a STALE
fdc0:2::216:3eff:fe53:9731 lladdr 00:16:3e:53:97:31 STALE
fe80::216:3eff:fe53:9731 lladdr 00:16:3e:53:97:31 STALE
etu@machine1:~$ sudo ovs-appctl fdb/show C-3PO
 port  VLAN  MAC                Age
    1    10  d2:5a:97:08:02:4a  175
    2    10  00:16:3e:42:da:1a  175
    4    10  00:16:3e:72:2b:8d  175
    3    10  00:16:3e:53:97:31  175

```
