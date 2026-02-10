---
author: Rui Wei
pubDatetime: 2026-02-10T20:35:41+08:00
modDatetime: 2026-02-10T20:36:32+08:00
title: OpenVPN 3
slug:
featured: false
draft: true 
tags: [linux, network, security]
description: OpenVPN 3
imageNameKey: openvpn_3
csl: vancouver
---


## Table of contents


## Introduction
OpenVPN 3 Linux client is the successor to OpenVPN 2 which no longer follows the single monolith design but instead spawn multiple daemons to handle the OpenVPN protocol. This blog serves as an introduction to the OpenVPN protocol and how it works.

Let's first give an introduction to how OpenVPN 3 operate at a higher level. You can think of OpenVPN 3 as an orchestration system that manages client VPN session as a managed object that is handled across multiple daemons. The diagram below shows the high-level architecture when a client initiates a VPN session.

```bash
+--------------------+
| openvpn3 CLI / UI |
+--------------------+
           |
           | D-Bus (control)
           v
+--------------------------+
| openvpn3-service-backend|
| (Session orchestration) |
+--------------------------+
           |
           | spawns
           v
+--------------------------+
| openvpn3-service-client |
| (TLS, auth, control)    |
+--------------------------+
           |
           | configures
           v
+--------------------------+
| openvpn3-service-netcfg |
| (Routes, DNS, TUN)      |
+--------------------------+
           |
           | kernel
           v
+--------------------------+
| TUN interface (tun0)    |
+--------------------------+
```

`openvpn3-service-backend`, `openvpn3-service-client` and `openvpn3-service-netcfg`  are examples of sub daemons that will interact with each other to manage a client VPN session. This is supported through a system called D-bus. D-bus provides a standardized and high-level way for processes to communicate to each other through service discovery and structured messages.  Firstly, when the client initiates a session via the CLI, it interacts with its sub daemons to keep track of the client VPN session, to do TLS handshake with the VPN server and  to create a tunnel for the client to communicate privately.

Let's start from the top. The client can initiate a VPN session by providing a configuration file (`ovpn`) through this command: `openvpn3 session-start --config client.ovpn`. The configuration file tells the OpenVPN client where to look for the VPN server, how to authenticate and encrypt the traffic, and how to route the traffic once it is connected. Here is an example of such a file.
```bash
client # identify as a client
dev tun # creates a layer 3 ip tunnel
remote vpn.example.com # vpn server hostname or ip
resolv-retry infinite # keep trying to resolve if DNS fails
nobind # tells client not to bind to a certain port
persist-key
persist-tun 
tls-client # use tls
verb 1 # logging verbosity 
keepalive 10 120 # send ping every 10s, 120s no response then reconnect
port 1194 # server port
proto udp # transport layer for tunnel
cipher BF-CBC # encryption algo
comp-lzo # compression to improve speed
remote-cert-tls server 

route 10.0.0.0 255.0.0.0 # route 10.0.0.0/8 through the tunnel
dhcp-option DNS 10.8.0.1 # the dns server in the private network

<ca> # the certificate authority that you trust
-----BEGIN CERTIFICATE-----
MIIE1jCCA76gAwIBAgIJAOMAQRbD8ADYMA0GCSqGSIb3DQEBCwUAMIGiMQswCQYD
...
xcPC3D4Gk0EW83PJorGi1+lPGNusEDO0xqlv2pLyQ07XVKWsYZo3AKQY
-----END CERTIFICATE-----
</ca>

cert /path/to/cert/file.crt # client's identity certificate
key /path/to/key/file.key # client's private key that will be use for signing

<tls-auth> # optional but does HMAC auth
-----BEGIN OpenVPN Static key V1-----
073b0025464cdeaa6189247397d0f2f6
...
9a9a92359aa0574a95715a1df0e51484
-----END OpenVPN Static key V1-----
</tls-auth>

```

Once that happens, a session object is created that gets managed by a daemon in Open VPN 3. Afterwards, the client initiates a TLS handshake - typically Ephemeral Diffie-Hellman - to get a session key that will be used with the cipher to encrypt the message. Do note that these keys are not actually used to transmit data content. Instead, once the TLS session is established, the VPN server will assign the client a private IP like `10.8.0.6` and treat the client as if the device is within the virtual private cloud. Other information like route prefix, DNS settings, MTU and other parameters are also forwarded to the client safely across TLS.

Now the client will attempt to create a tunnel, *TUN* device: 
```bash
/dev/net/tun
→ tun0 (or ovpn0)
```
`tun0` is actually a virtual network interface that the OS will recognize now. So if we do `ip a` 

```bash
..
```


The client will also then modify the routing table to install routes such that the kernel will then forward those IP packets to the virtual network interface instead of the physical network card.
```bash
...

```


So the entire flow can be summarized as such. We can basically treat Open VPN 3 as an intermediary that intercepts message that are intended to be encrypted just before it goes out from our network card.
```bash
Message -> tun0 -> OpenVPN3 -> Physical NIC -> VPN Server Public IP
```


## DNS Integration
Let's do a little recap, suppose we want to `ping www.google.com`, how do we find out the IP address? We have to contact a DNS server to resolve the domain name to an IP address but where can we find the DNS server? Typically, after DHCP - a service that allocates you an IP address - the daemon will fill in `/etc/resolv.conf` with DNS server information. Nowadays, there is a system service, `systemd-resolved` that handles DNS queries for us. In short, DHCP would typically contact that service to update, if any, DNS server IP addresses. Now in `/etc/resolv.conf` you would find `nameserver 127.0.0.53` which is a loop back address so that `systemd-resolved`  would be the one handling the DNS query.

- talk a little about DNS
- give a real life example video of aws virtual private network cloud

- talk a bit about the differerence betewen this and ipsec vpn and maybe give some insights on why not just use TLS entirely




https://gist.github.com/renatolfc/f6c9e2a5bd6503005676

https://medium.com/@linuxrootroom/dd-721dc2b25d1c