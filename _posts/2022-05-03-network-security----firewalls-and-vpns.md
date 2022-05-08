---
layout: post
title: "Network Security - Firewalls and VPNs"
subtitle: "Network Security Labs"
date: 2022-05-03
author: "Bingkun"
header-img: "img/post-bg-cyber-security.png"
tags: [Network, Security]
---

### Overview
- Our friend Good Guy owns teo servers
    - **vulnerable**: running a service that's interesting for the world
    - **secret**: where he keeps some important secret he needs
- The attacker Kali has heard of a valuable secret, and wants to find it!
- But Good Guy will secure his network with a firewall and a VPN
    - The firewall will protect his secret from the world
    - The VPN will allow him (and only him) to access the secret through the firewall
    - Or so he thinks...

### Gateway Configuration
<img src="/img/in-post/post-vpn-firewall/environment.png" alt="environment" width="800"/>
Configure the gateway such that it forwards traffic between Network A and Network B.
```
sysctl -w net.ipv4.ip_forward=1
```
Change the default gateway on all machines to the address of the gateway machine.

On all machines (except gateway):
```
sudo route del default
```
On secret and vulnerable:
```
sudo route add default gw 192.168.56.101
```
The route table will be like:
<img src="/img/in-post/post-vpn-firewall/service-route.png" alt="environment" width="800"/>

On kali and goodguy:
```
sudo route add default gw 10.0.10.101
```
The route table will be like:
<img src="/img/in-post/post-vpn-firewall/user-route.png" alt="environment" width="800"/>

### VPN
#### VPN Overview
<img src="/img/in-post/post-vpn-firewall/vpn-config.png" alt="vpn-config" width="800"/>
> - Setup VPN connection with OpenVPN
> - It should create a tunnel between goodguy and the 192.168.56.0/24 subnet
> - gateway shall be the OpenVPN server and Certificate Authority (CA)
> - It shall generate and sign certificates for gateway (server) and goodguy (client)
> - It should also generate all the cryptographic RSA key pairs, even the one for goodguy

On gateway, we copy all necessary files to a single convenient folder:
```
Become root: sudo su (to avoid having to type sudo so much)
cp -r /usr/share/doc/openvpn/ /etc/openvpn/
cp -r /usr/share/easy-rsa/* /etc/openvpn/
cd /etc/openvpn
```

Then, we generate all the certificates and diffie-hellman parameters:
```
mv openssl-1.0.0.cnf openssl.cnf
source ./vars
./clean-all
./build-ca
./build-key-server server
./build-dh
./build-key goodguy
```

We then send the required files from gateway to goodguy:
**On goodguy**
```
nc -l -p 1337 > certs.tar.gz #Do this before sending it on gateway
```
**On gateway** 
```
cd keys
```
**On gateway**
```
tar -cz ca.crt goodguy.crt goodguy.key | nc -w 1 10.0.10.103 1337
```
**On goodguy**
```
tar -xvf certs.tar.gz
```

Now we unzip the server.conf and update the client.conf files for OpenVPN:
**On gateway**
```
cd keys
gunzip ../openvpn/examples/sample-config-files/server.conf.gz
Comment out the ta.key in ../openvpn/examples/sample-config-files/server.conf
# tls-auth ta.key 0
openvpn ../openvpn/examples/sample-config-files/server.conf #Start the VPN server
```
**On goodguy**
```
Move the client conf to a local directory
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ./
Change the remote line to remote 10.0.10.101 1194
Set the ca , cert and key parameters to the respective files that we received from gateway and
comment out the ta.key here as welll
Start the VPN client sudo openvpn client.conf
```
Open a new terminal on goodguy.


reroute our traffic through the VPN:
```
sudo route add -net 192.168.56.0/24 gw 10.8.0.5
```
And check that we are being routed through the VPN:
```
raceroute 192.168.56.102
```
<img src="/img/in-post/post-vpn-firewall/traceroute.png" alt="traceroute" width="800"/>

### Firewall
Configure gateway to work as a firewall.

Allow traffic to/from vulnerable and block everything else:
```
sudo iptables -A FORWARD -d 192.168.56.103 -j ACCEPT
sudo iptables -A FORWARD -s 192.168.56.103 -j ACCEPT
sudo iptables -P FORWARD DROP
```

verify by using ping from kali to secret and vulnerable:
```
ping 192.168.56.102 -> results in error
ping 192.168.56.103 -> successful
```

Configure gateway to allow goodguy to access secret through the VPN:
```
sudo iptables -A FORWARD -i tun0 -o eth1 -j ACCEPT
sudo iptables -A FORWARD -o tun0 -i eth1 -j ACCEPT
```

goodguy should now be able to access the secret machine:
```
curl 192.168.56.102 -> completes successfully!
```

## Use Your Evil Power
Use the nmap tool to scan the vulnerable machine and identify possible vulnerable services.

Decoding the hint, we wonder if the machine is vulnerable to Shellshock through the Apache cgi-bin
attack vector?
```
sudo nmap -T5 -sV –script http-shellshock –script-args uri=/cgi-bin/service.sh -v
192.168.56.103
```

