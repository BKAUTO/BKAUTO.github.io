---
layout: post
title: "Network Security - IPsec VPNs"
subtitle: "Network Security Labs"
date: 2022-05-15
author: "Bingkun"
header-img: "img/post-bg-cyber-security.png"
tags: [Network, Security]
---

This homework assignment has two parts: in the first part, you will use the ipip module to create an IP tunnel
between two computers; in the second part, you will use IPsec to encrypt the tunnel, effectively creating a VPN.

For this assignment you will need a local network of two Linux computers. You may, of course, use two VMs. (In
fact, the provided solutions are based on two Ubuntu VMs.)

### IP Tunneling and IPsec
Load in the linux kernel the ipip module for IP tunnelling. If this is pre-installed in your distribution, you can
simply load it with:
```
sudo modprobe ipip
```
Download and Install ``ipsec-tools``
```
wget http://mirrors.kernel.org/ubuntu/pool/universe/i/ipsec-tools/ipsec-tools_0.8.2+20140711-10build1_amd64.deb
sudo dpkg -i ./ipsec-tools_0.8.2+20140711-10build1_amd64.deb
```
Create a point-to-point LAN between two computers, Alice and Bob. This LAN will play the role of the untrusted
network.
```
Alice has IP 192.168.56.20 on interface eth1.
Bob has IP 192.168.56.40 on interface eth1.
```
Use the tool ip to create an IPIP tunnel, named tun0, between Alice and Bob.
The tunnel will be a virtual LAN on top of LAN that you created in the previous step.
```
On Alice, we create the tunnel using the following command:
$ sudo ip tunnel add tun0 mode ipip remote 192.168.100.2 local 192.168.100.1
Similarly on Bob:
$ sudo ip tunnel add tun0 mode ipip remote 192.168.100.1 local 192.168.100.2
```
This command creates an ipip tunnel named tun0 between Alice (192.168.56.20) and Bob (192.168.56.40)
respectively.

We can run ifconfig to confirm that the tunnel is created:
```
ifconfig -a tun0
tun0: flags=144<POINTOPOINT,NOARP>  mtu 1480
        tunnel   txqueuelen 1000  (IPIP Tunnel)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
The second step is to assign IP to the tunnel. Let’s say that the tunnel will be on the subnet 10.0.0.0/24
and Alice will have the IP 10.0.0.1 and Bob the IP 10.0.0.2.
```
On Alice:
$ sudo ifconfig tun0 10.0.0.1/24 up
On Bob:
$ sudo ifconfig tun0 10.0.0.2/24 up
```
You may use ifconfig and route to confirm that everything is as it should be. You can also observe that the
virtual interface tun0 appears to be identical to other physical interfaces (e.g. eth0).
```
tun0: flags=209<UP,POINTOPOINT,RUNNING,NOARP>  mtu 1480
        inet 10.0.0.1  netmask 255.255.255.0  destination 10.0.0.1
        inet6 fe80::5efe:c0a8:3814  prefixlen 64  scopeid 0x20<link>
        tunnel   txqueuelen 1000  (IPIP Tunnel)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 6  dropped 0 overruns 0  carrier 0  collisions 0
```
Finally, we ping from Bob to Alice to confirm that the communication works fine.
```
ping 10.0.0.1
```
On Alice, we set Wireshark to capture traffic on the interface of the LAN, which is enp0s8 in my VM.

You should observe that there are two IP headers on the ICMP packets. The inner IP header has the source
and destination IP of the tunnel (10.0.0.1 and 10.0.0.2 respectively), while the outer IP header has the
actual source and destination IP (192.168.56.20 and 192.168.56.40 respectively).

In practice, the ICMP packet that is generated in Bob uses the tunnel IP addresses as if the tunnel was a
LAN. When it reaches the tunnel, an additional header is attached with the real IP addresses, which in our
case are local IP addresses, but in reality they can be public IP addresses. The packet is then routed normally
(potentially over the Internet) based on the outer IP header, until it reaches the destination IP address. Over
there, the outer IP header is removed and the packet is delivered to the other tunnel end.

If you repeat the process with Wireshark on tun0, you will observe that the packets appear with just one
IP header. The tunnel, in practice, abstracts the IP over IP operation. From the perspective of tun0, the
connection appears to be a normal LAN, as usual. In this example we created a tunnel over a LAN, but in a
real-world scenario, we could have created a tunnel over two remote places of the Internet, and these remote
locations would behave as if they were in a local network. This is the basic principle of VPNs.

You should also notice that the ICMP packets are still visible to an eavesdropper of the physical interface
(such as Wireshark on enp0s8). A tunnel is a necessary step but it does not provide security. We will need to
add encryption to create a VPN.

### IPsec
Let us now add encryption on the tunnel (tun0) using IPsec.

Create two IPsec configuration files: ipsec_alice.conf and ipsec_bob.conf, respectively.

The configuration file of Alice looks like that:
```
$ cat ipsec_alice.conf
# Flush the SAD and SPD
flush;
spdflush;
# ESP SAs using 192 bit long keys (168 + 24 parity)
add 10.0.0.1 10.0.0.2 esp 0x201 -E 3des-cbc
0x7aeaca3f87d060a12f4a4487d5a5c3355920fae69a96c831;
add 10.0.0.2 10.0.0.1 esp 0x301 -E 3des-cbc
0x7aeaca3f87d060a12f4a4487d5a5c3355920fae69a96c831;
# Security policies
spdadd 10.0.0.1 10.0.0.2 any -P out ipsec
esp/transport//require;
spdadd 10.0.0.2 10.0.0.1 any -P in ipsec
esp/transport//require;
```
The configuration file for Bob looks like that:
```
$ cat ipsec_bob.conf
# Flush the SAD and SPD
flush;
spdflush;
# ESP SAs using 192 bit long keys (168 + 24 parity)
add 10.0.0.2 10.0.0.1 esp 0x301 -E 3des-cbc
0x7aeaca3f87d060a12f4a4487d5a5c3355920fae69a96c831;
add 10.0.0.1 10.0.0.2 esp 0x201 -E 3des-cbc
0x7aeaca3f87d060a12f4a4487d5a5c3355920fae69a96c831;
# Security policies
spdadd 10.0.0.2 10.0.0.1 any -P out ipsec
esp/transport//require;
spdadd 10.0.0.1 10.0.0.2 any -P in ipsec
esp/transport//require;
```
We may now load them on Alice:
```
$ sudo setkey -f ipsec_alice.conf
```
And Bob:
```
$ sudo setkey -f ipsec_bob.conf
```
Open Wireshark (or tshark) on the physical LAN interface (not tun0).

In turn, ping Alice from Bob.

You will now notice that the content of the IP packet on the tunnel (ICMP packet) is now encrypted and
Wireshark displays them as ESP.

You have effectively added security on the network layer. IPsec encrypts the data, and even if the application
(or ping in our example) does not do any encryption whatsoever, the confidentiality of the data is protected
by the network.

