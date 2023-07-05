---
layout: post
title: "Network Security - Transport Layer Security"
subtitle: "Network Security Labs"
date: 2022-05-13
author: "Bingkun"
header-img: "img/post-bg-cyber-security.png"
tags: [Network, Security]
---

### SSL/TLS
The goal of this assignment is to set up a secure server that uses SSL/TLS. For that you will need two Linux
computers (or VMs) that are connected: Alice will play the role of the client and Bob will play the role of the
server.

Begin by setting up IP addresses for Alice and Bob and confirming that they can ping each other.
```
vagrant@alice:~$ ping 192.168.56.40
PING 192.168.56.40 (192.168.56.40) 56(84) bytes of data.
64 bytes from 192.168.56.40: icmp_seq=1 ttl=64 time=0.400 ms
64 bytes from 192.168.56.40: icmp_seq=2 ttl=64 time=0.516 ms
--- 192.168.56.40 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1023ms
rtt min/avg/max/mdev = 0.400/0.458/0.516/0.058 ms
```
```
vagrant@bob:~$ ping 192.168.56.20
PING 192.168.56.20 (192.168.56.20) 56(84) bytes of data.
64 bytes from 192.168.56.20: icmp_seq=1 ttl=64 time=0.373 ms
64 bytes from 192.168.56.20: icmp_seq=2 ttl=64 time=0.445 ms
--- 192.168.56.20 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1025ms
rtt min/avg/max/mdev = 0.373/0.409/0.445/0.036 ms
```
Use openssl genrsa to generate an RSA private key of 1024 bits.
```
openssl genrsa -out private.key 1024
```
Use the tool openssl req to generate a certificate using the RSA key that you created in the previous step. The
certificate links Bob’s identity to the RSA key. For this example, populate the certificate fields (country, address,
etc) with random data.
```
openssl req -new -key private.key -out req.csr
```
Normally, Bob would send the CSR to a trusted Certification Authority (CA) that would generate the certificate
by signing it with their private key.
In this assignment, instead, you are asked to generate a self-signed certificate. In other words, you are asked to
generate a certificate that is signed using Bob’s private key.
Use the tool openssl x509 to sign the CSR that you generated in the previous step. Set the certificate expiration
date in one year.
```
openssl x509 -req -days 365 -in req.csr -signkey private.key -out certificate.pem
```
Note that this has little value in a practical setting, as it is equivalent to a person issuing her own passport.
Hence, if you try to use this certificate in an HTTP server, the browser will issue a warning that the certificate
is not trustworthy. However, self-signed certificates are good for testing and experimenting purposes.

Use openssl x509 to inspect the details of the certificate.
```
openssl x509 -text -in certificate.pem
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 12225999660122272321 (0xa9ab79ea2da27241)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: C=DK, ST=Copenhagen, L=Lyngby, O=DTU, OU=Compute
        Validity
            Not Before: May 13 14:17:43 2022 GMT
            Not After : May 13 14:17:43 2023 GMT
        Subject: C=DK, ST=Copenhagen, L=Lyngby, O=DTU, OU=Compute
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (1024 bit)
                Modulus:
                    00:c5:2b:9f:8c:c8:c5:89:9d:bf:59:c3:e4:aa:44:
                    0f:cd:e2:c0:7f:d0:1c:1c:c6:a4:ba:cc:d1:36:13:
                    cf:97:3d:c1:9b:63:7f:b9:2c:3e:60:99:7c:32:62:
                    b8:51:73:5e:56:62:d9:53:ec:7a:4b:7f:8d:bf:33:
                    c7:99:89:5b:57:e7:9f:e0:59:b0:94:17:71:df:6b:
                    ea:8f:bf:dd:f9:53:72:14:04:a7:04:d4:bf:d9:8f:
                    bb:29:39:25:b8:71:e7:63:31:2b:fd:80:9d:be:90:
                    80:6e:14:1c:c4:b9:8e:81:a4:d6:f4:6c:ab:3a:05:
                    69:29:10:7a:82:eb:e1:28:19
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha1WithRSAEncryption
         87:4d:27:11:ad:c7:34:62:2f:52:57:02:27:1e:20:7f:be:61:
         87:37:69:51:91:5c:b5:b3:d9:e7:ab:80:ae:eb:56:99:ad:67:
         dd:2b:8b:c0:a6:77:c0:6b:67:32:83:9e:07:a8:02:d0:af:c1:
         76:cd:4a:df:77:dd:33:ca:bf:c0:af:6b:d7:00:6f:14:33:5c:
         7c:98:08:40:98:23:2c:f1:03:62:e1:7a:ba:65:f8:7b:85:fc:
         96:5c:44:91:c8:98:2f:ee:ba:84:17:c3:c8:f9:85:5c:24:32:
         e9:89:f5:ec:24:67:3c:d5:23:ef:af:8a:66:6c:44:24:f7:f1:
         86:1a
-----BEGIN CERTIFICATE-----
MIICHTCCAYYCCQCpq3nqLaJyQTANBgkqhkiG9w0BAQUFADBTMQswCQYDVQQGEwJE
SzETMBEGA1UECAwKQ29wZW5oYWdlbjEPMA0GA1UEBwwGTHluZ2J5MQwwCgYDVQQK
DANEVFUxEDAOBgNVBAsMB0NvbXB1dGUwHhcNMjIwNTEzMTQxNzQzWhcNMjMwNTEz
MTQxNzQzWjBTMQswCQYDVQQGEwJESzETMBEGA1UECAwKQ29wZW5oYWdlbjEPMA0G
A1UEBwwGTHluZ2J5MQwwCgYDVQQKDANEVFUxEDAOBgNVBAsMB0NvbXB1dGUwgZ8w
DQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBAMUrn4zIxYmdv1nD5KpED83iwH/QHBzG
pLrM0TYTz5c9wZtjf7ksPmCZfDJiuFFzXlZi2VPsekt/jb8zx5mJW1fnn+BZsJQX
cd9r6o+/3flTchQEpwTUv9mPuyk5Jbhx52MxK/2Anb6QgG4UHMS5joGk1vRsqzoF
aSkQeoLr4SgZAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAh00nEa3HNGIvUlcCJx4g
f75hhzdpUZFctbPZ56uArutWma1n3SuLwKZ3wGtnMoOeB6gC0K/Bds1K33fdM8q/
wK9r1wBvFDNcfJgIQJgjLPEDYuF6umX4e4X8llxEkciYL+66hBfDyPmFXCQy6Yn1
7CRnPNUj76+KZmxEJPfxhho=
-----END CERTIFICATE-----
```
Use the tool openssl s_server to set up a test web server on port 443. Use the certificate of Bob (which contains
his public key) and the private key of Bob.

The openssl s_server tool can create a test web server using the -www flag. This server will respond with
a status page. Alternatively, you may also use the -WWW flag to host html files.

We also use the HTTPS default port 443.
```
sudo openssl s_server -accept 443 -cert certificate.pem -www -key private.key
```
Let us now access the server from Alice.

On Alice, open Firefox and access the website: ``https://192.168.56.40/``

Observe that Firefox returns an error: Your connection is not secure.

Add an exception to Firefox and try again.

Open Wireshark (or tshark) and repeat the HTTPS request above (refresh the page).

Which version of TLS was used? Which symmetric encryption/authentication algorithm was used upon the TLS
negotiation between Alice and Bob?

We can observe the after the TCP handshake (SYN, SYN/ACK, ACK), the TSL handshake follows.

During the TSL handshake, keys were exchanged and a cipher algorithm was selected. The TSL version and
encryption algorithms can be seen in the TSL headers. In my setting, I can see the following:
```
Version: TLS 1.2 (0x0303)
Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)
```

The HTTP request and response are encrypted in TLS packets that are marked as Application Data. Different
to plain HTTP (see Assignment 3.1) the data are not accessible to an eavesdropper.