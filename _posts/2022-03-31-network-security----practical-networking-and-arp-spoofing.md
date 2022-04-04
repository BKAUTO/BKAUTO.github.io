---
layout: post
title: "Network Security - Practical Networking and ARP spoofing"
subtitle: "Network Security Labs"
date: 2022-03-31
author: "Bingkun"
header-img: "img/post-bg-cyber-security.png"
tags: [Network, Security]
---

### Goals

Alice, Bob and Kali, are on the same network and they can access the company webserver
(also on the same local network). Kali decides to perform a MITM attack on Alice! Kali will
use a cool hacking tool called Ettercap, which is able to perform arp spoofing and MITM
attacks. When successful Kali will be able to sniff (eavesdrop) all the traffic from Alice to the
webserver. But Kali wants to have more fun. Next step is to start a webserver (apache2) on
her machine which will display a really scary webpage! The goal is whenever Alice tries to view
the company’s page, she would be redirected instead to Kali’s scary webpage. She will go
crazy when she sees it!

### VM Setup and IP Config

<img src="/img/in-post/post-arp-spoof/ARP-VM.png" alt="Virtual Machine and IP configuration" width="400"/>

#### Virtual Machine
```
vagrant up
vagrant ssh alice
vagrant ssh bob
vagrant ssh webserver
```

#### Network Setup
Assign ip address to each of the machines with the following command:
```
sudo ip addr add 192.168.56.10X/24 dev eth1
```
Use ping to confirm that all hosts can reach each other.
For example, ping the webserver from Alice:
```
ping 192.168.56.110
```

#### Start Webserver on Kali
Kali now wants to run a webserver (Apache2) for spoofing.
First edit the webpage the server will host.
```
sudo vim /var/www/html/index.html
```
Change it into something scary.
```
<html><body>
<h1>Hello, this is your evil twin</h1>
<img src="https://i.imgur.com/ZpUjvbI.jpg" />
</body></html>
```
Start the webserver.
```
sudo service apache2 start
```
Finally, we can `curl` the webserver from Alice.
```
vagrant@alice:~$ curl 192.168.56.103
<html>
    <body>
        <h1>Hello, this is your evil twin</h1>
        <img src="https://i.imgur.com/ZpUjvbl.jpg"/>
    </body>
</html>
```
#### APR Spoofing
##### Start Wireshark on Kali
`curl` from Kali to the webserver. Can you see the HTTP packets?
Select the corresponding packets in Wiresharks.
```
ip.addr == 192.168.56.110
```
The packets for a HTTP GET can be seen.
<img src="/img/in-post/post-arp-spoof/keli-web.png" alt="keli-werver packets" width="800"/>
`curl` from Alice to the webserver. Can you see them now?

No.

##### Start ettercap on the Kali host
```
sudo ettercap -G
```
Scan for hosts on the eth1 interface. How is Ettercap able to find all hosts on the network?
<img src="/img/in-post/post-arp-spoof/hosts.png" alt="host list" width="300"/>
Ettercap broadcasts ARP packets to the whole LAN to find out if there is response for each netmask.
 <img src="/img/in-post/post-arp-spoof/host-scan.png" alt="host list" width="800"/>
 Set Alice as target1. Set webserver as target2.

 Click on MITM Menu, and select ARP poisoning from the drop down menu.

 On the MITM Attack dialogue window, leave the default settings and click OK.

 Alright. Ettercap is now ready to perform the MitM ARP poisoning attack on the specified targets in the network. 

 Before we perform the ARP poisoning attack, let’s look at the ARP information on Alice.
 ```
vagrant@alice:~$ arp -a
? (192.168.56.103) at 08:00:27:9e:01:8c [ether] on eth1
? (192.168.56.110) at 08:00:27:ba:2a:ed [ether] on eth1
 ```
 Let's start ARP poisoing and check the ARP information again on Alice.
  <img src="/img/in-post/post-arp-spoof/sniff.png" alt="start sniffing" width="300"/>
 ```
vagrant@alice:~$ arp -a
? (192.168.56.103) at 08:00:27:9e:01:8c [ether] on eth1
? (192.168.56.110) at 08:00:27:9e:01:8c [ether] on eth1
 ```
 The MAC address of the Web Server before and during the ARP poisoning attack changed. Alice thinks it is still talking with the Web Server when a MitM attack compromises the connection. 

#### Redirect to Evil website
Alice still sees the genuine website from the webserver but is being redirected through the Kali machine.
```
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -d 192.168.56.110 -j DNAT --to-destination 192.168.56.103
```
Now, `curl` Web Server from Alice.
```
vagrant@alice:~$ curl 192.168.56.110
<html>
    <body>
        <h1>Hello, this is your evil twin</h1>
        <img src="https://i.imgur.com/ZpUjvbl.jpg"/>
    </body>
</html>
```
Scary! Isn't it?





