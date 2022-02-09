---
layout: post
title: "Network Security - Network Essentials I"
subtitle: "Network Security Labs"
date: 2022-02-09
author: "Bingkun"
header-img: "img/post-bg-cyber-security.png"
tags: [Network, Security]
---

### 1. GBAR – TCP/IP Settings and Configuration
##### Ifconfig
First we connect to the DTU HPC.
```
ssh <username>@login.gbar.dtu.dk
```
Then use the <mark>ifconfig</mark> to inspect the networking interfaces of **login.gbar.dtu.dk**.
```
gbarlogin1(*******) $ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.66.10.34  netmask 255.255.224.0  broadcast 10.66.31.255
        inet6 fe80::627:58ff:fe0a:fa3f  prefixlen 64  scopeid 0x20<link>
        ether 04:27:58:0a:fa:3f  txqueuelen 1000  (Ethernet)
        RX packets 824172505  bytes 680116355322 (633.4 GiB)
        RX errors 0  dropped 0  overruns 162  frame 0
        TX packets 524326815  bytes 395556339336 (368.3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0xc8100000-c81fffff

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.38.95.134  netmask 255.255.255.0  broadcast 192.38.95.255
        inet6 2001:878:200:3010:dcc:3141:5926:5134  prefixlen 64  scopeid 0x0<global>
        inet6 fe80::627:58ff:fe0a:fa40  prefixlen 64  scopeid 0x20<link>
        ether 04:27:58:0a:fa:40  txqueuelen 1000  (Ethernet)
        RX packets 492578391  bytes 262881812906 (244.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 601959955  bytes 798601549920 (743.7 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device memory 0xc8000000-c80fffff

ib0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 2044
        inet 10.66.80.44  netmask 255.255.240.0  broadcast 10.66.95.255
        inet6 fe80::22f1:7c03:c4:adfa  prefixlen 64  scopeid 0x20<link>
        inet6 fd23:711a:2e3c:49e5:22f1:7c03:c4:adfa  prefixlen 64  scopeid 0x0<global>
Infiniband hardware address can be incorrect! Please read BUGS section in ifconfig(8).
        infiniband 80:00:02:08:FE:80:00:00:00:00:00:00:00:00:00:00:00:00:00:00  txqueuelen 256  (InfiniBand)
        RX packets 8825440  bytes 518586377 (494.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 276379  bytes 24184534 (23.0 MiB)
        TX errors 0  dropped 4 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 47305683  bytes 311245231720 (289.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 47305683  bytes 311245231720 (289.8 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
There are 2 Ethernet interfaces, **eth0** and **eth1**.

**eth0** has a IPV4 address of 10.66.10.34. This is a private IP address which is in the range 10.0.0.0 to 10.255.255.255. This address is not publicly routable.

```
inet 192.38.95.134  
netmask 255.255.255.0  
broadcast 192.38.95.255
inet6 2001:878:200:3010:dcc:3141:5926:5134
```
Above shows the IP addresses of interface **eth1**.

The MAC address (hardware address) of **eth0** is: 
```
ether 04:27:58:0a:fa:3f
```

**eth0** has a IPv4 address of 10.66.10.34 and the netmask is 255.255.224.0. As a result, $2^{13} - 2 = 8190$ hosts is available for this same network.

##### route

Then we use <mark>route</mark> to inspect the routing table of **login.gbar.dtu.dk**.
```
gbarlogin1(*******) $ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.38.95.1     0.0.0.0         UG    0      0        0 eth1
10.66.0.0       0.0.0.0         255.255.224.0   U     0      0        0 eth0
10.66.80.0      0.0.0.0         255.255.240.0   U     0      0        0 ib0
192.38.95.0     0.0.0.0         255.255.255.0   U     0      0        0 eth1
239.2.11.71     0.0.0.0         255.255.255.255 UH    0      0        0 eth0
```
The IP address of the default gateway is <mark>192.38.95.1</mark>. The traffic directed to the default gateway is routed to interface **eth1**.

##### Ping
Use ping to exchange ICMP messages with other hosts.

Ping the git server of DTU Compute (git.compute.dtu.dk) and inspect the statistics of the round-trip time (rtt).
```
gbarlogin1(*******) $ ping git.compute.dtu.dk -c 4
PING git.compute.dtu.dk (130.225.68.228) 56(84) bytes of data.
64 bytes from git.compute.dtu.dk (130.225.68.228): icmp_seq=1 ttl=60 time=0.454 ms
64 bytes from git.compute.dtu.dk (130.225.68.228): icmp_seq=2 ttl=60 time=0.415 ms
64 bytes from git.compute.dtu.dk (130.225.68.228): icmp_seq=3 ttl=60 time=0.509 ms
64 bytes from git.compute.dtu.dk (130.225.68.228): icmp_seq=4 ttl=60 time=0.448 ms

--- git.compute.dtu.dk ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.415/0.456/0.509/0.039 ms
```
Ping the git server of GitHub (git.github.com) and inspect the rtt statistics. 
```
gbarlogin1(*******) $ ping git.github.com -c 4
PING github.github.io (185.199.108.153) 56(84) bytes of data.
64 bytes from cdn-185-199-108-153.github.com (185.199.108.153): icmp_seq=1 ttl=55 time=10.0 ms
64 bytes from cdn-185-199-108-153.github.com (185.199.108.153): icmp_seq=2 ttl=55 time=9.97 ms
64 bytes from cdn-185-199-108-153.github.com (185.199.108.153): icmp_seq=3 ttl=55 time=9.93 ms
64 bytes from cdn-185-199-108-153.github.com (185.199.108.153): icmp_seq=4 ttl=55 time=10.1 ms

--- github.github.io ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 9.939/10.026/10.178/0.091 ms
```
The git server of Github has longer delay.

##### Host
Use host to find the IP address that is associated to a host name.

What is the IP address of the git server of DTU Compute (git.compute.dtu.dk)?
```
gbarlogin1(*******) $ host git.compute.dtu.dk
git.compute.dtu.dk has address 130.225.68.228
```
What is the IP address of the server th11.gbar.dtu.dk?
```
gbarlogin1(*******) $ host th11.gbar.dtu.dk
th11.gbar.dtu.dk has address 192.38.95.88
th11.gbar.dtu.dk has IPv6 address 2001:878:200:3010:dcc:3141:5926:5388
```
What is the IP address of one of the git servers of GitHub (git.github.com)?
```
gbarlogin1(*******) $ host git.github.com
git.github.com is an alias for github.github.io.
github.github.io has address 185.199.109.153
github.github.io has address 185.199.111.153
github.github.io has address 185.199.110.153
github.github.io has address 185.199.108.153
github.github.io has IPv6 address 2606:50c0:8002::153
github.github.io has IPv6 address 2606:50c0:8000::153
github.github.io has IPv6 address 2606:50c0:8001::153
github.github.io has IPv6 address 2606:50c0:8003::153
```
Is the server <mark>th0.gbar.dtu.dk</mark> at the same network (subnet) with <mark>login.gbar.dtu.dk</mark>?
```
gbarlogin1(*******) $ host th0.gbar.dtu.dk
th0.gbar.dtu.dk has address 192.38.95.89
th0.gbar.dtu.dk has IPv6 address 2001:878:200:3010:dcc:3141:5926:5389

gbarlogin1(*******) $ host login.gbar.dtu.dk
login.gbar.dtu.dk is an alias for login1.gbar.dtu.dk.
login1.gbar.dtu.dk has address 192.38.95.134
login1.gbar.dtu.dk has IPv6 address 2001:878:200:3010:dcc:3141:5926:5134
```
The subnet mask is 255.255.255.0, so they are in the same subnet.

##### ARP
Use <mask>arp</mark> to inspect the ARP table of <mark>login.gbar.dtu.dk</mark>

What is the MAC address of the server <mark>th0.gbar.dtu.dk</mark>?
```
gbarlogin1(*******) $ arp th0.gbar.dtu.dk
Address                  HWtype  HWaddress           Flags Mask            Iface
th0.gbar.dtu.dk          ether   24:44:27:eb:4f:22   C
```

##### Traceroute
How many routers are in the path between login.gbar.dtu.dk and www.dtu.dk (the web server of DTU)?
```
gbarlogin1(*******) $ traceroute www.dtu.dk
traceroute to www.dtu.dk (192.38.84.35), 30 hops max, 60 byte packets
 1  gbar.net.dtu.dk (192.38.95.1)  0.721 ms  0.835 ms  0.972 ms
 2  10.12.0.41 (10.12.0.41)  0.381 ms 10.12.0.45 (10.12.0.45)  0.342 ms  0.554 ms
 3  192.38.93.227 (192.38.93.227)  0.542 ms  0.538 ms  0.519 ms
 4  vl1040.wd-ly202-9988-1.router.net.dtu.dk (192.38.93.209)  1.087 ms  1.119 ms  1.153 ms
 5  hu1_0_28.40.pir-ly202-9988-1.router.net.dtu.dk (192.38.93.123)  1.051 ms  1.047 ms  1.029 ms
 6  hu1_0_27.1120.wd-ly202-9988-1.router.net.dtu.dk (192.38.73.122)  1.094 ms  0.907 ms  0.874 ms
 7  192.38.73.107 (192.38.73.107)  1.165 ms  1.097 ms  1.113 ms
 8  vl1190.sd2-ly-304-m1-o.router.net.dtu.dk (192.38.73.114)  1.418 ms  1.602 ms  1.355 ms
 9  external8.sitecore.dtu.dk (192.38.84.35)  1.138 ms  1.119 ms  1.097 ms
```