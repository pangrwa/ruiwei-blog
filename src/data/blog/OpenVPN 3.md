---
author: Rui Wei
pubDatetime: 2026-02-10T20:35:41+08:00
modDatetime: 2026-02-10T20:36:32+08:00
title: OpenVPN 3
slug:
featured: false
draft: false
tags:
  - linux
  - network
  - security
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
cipher AES-256-GCM # encryption algo
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
3: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.8.0.2/24 brd 10.8.0.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::a1b2:c3d4:e5f6:7890/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
```
The private IP that is assigned to us within the virtual private cloud is `10.8.0.2/24`

The client will also then modify the routing table to install routes such that the kernel will then forward those IP packets to the virtual network interface instead of the physical network card. So if we do a `ip route`:
```bash
default via 192.168.1.1 dev eth0 proto dhcp metric 100 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100 metric 100
10.8.0.0/24 dev tun0 proto kernel scope link src 10.8.0.2 
10.50.0.0/16 via 10.8.0.1 dev tun0 
```
The first two rows are your untouched default route that basically goes through your original network card `eth0`. Now suppose `10.50.0.0/16` is whatever private IP that your internal network is using, any traffic going towards there would be forwarded to the `tun0` virtual interface which would be forwarded to the openvpn3 application.


So the entire flow can be summarized as such. We can basically treat Open VPN 3 as an intermediary that intercepts message that are intended to be encrypted just before it goes out from our network card.
```bash
Message -> tun0 -> OpenVPN3 -> Physical NIC -> VPN Server Public IP
```


## DNS Integration
Let's recap. Suppose we want to `ping www.google.com`, how do we find out the IP address? We have to contact a DNS server to resolve the domain name to an IP address but where can we find the DNS server? Typically, after DHCP - a protocol that allocates you an IP address. On older setups, the DHCP client might fill  in `/etc/resolv.conf` directly with DNS server information.
Nowadays, the default system and service manager for most Linux distributions, `systemd`, will manage a background service - either `NetworkManager` or `systemd-networkd` - that basically handles everything related to setting up the network. In this case, the NetworkManager will contain the additional information from the DHCP client and will then publish the DNS configuration to `systemd-resolved` via the D-Bus API. Now in `/etc/resolv.conf` you would find `nameserver 127.0.0.53` which is a loop back address that will be handled by `systemd-resolved`.

Suppose we `ping www.google.com`, `ping` binary would ask libc function, `getaddrinfo`, which will look in `/etc/resolv.conf` to find the appropriate DNS server to craft the DNS request message. `systemd-resolved` will be listening to that loop back interface and will then act as the intermediary that handles the DNS operation. 

So how does this tie in with OpenVPN3? OpenVPN3, interacts with `systemd-resolved` via the D-Bus API, telling it to use these DNS servers for these private IP addresses that it received from the server. For example, if your private domain is `corp.example`, `systemd-resolved` would automatically know which DNS server to contact to to get the proper private IP in the virtual private cloud. There are two common DNS patterns
1. **Full-tunnel DNS**: All DNS queries goes to the VPN-provided DNS server
2. **Split-DNS**: Filtered domains goes to the VPN-provided DNS server while the rest go to the local DNS server

You can run `resolvectl status` to view which is the default DNS server that each network interface contacts.

I refer an insightful and practical video of the relationship between OpenVPN 3 and AWS virtual private network cloud - click [here](https://www.youtube.com/watch?v=St8y0xZSn3c).


## OpenVPN vs IPsec
IPsec operates at the kernel level (Network Layer). So unlike OpenVPN where the message is modified on the user-space (application level) right when the packet leaves the virtual network card, IPsec jumps in directly when the OS wraps the IP packet. It notices that traffic destined to `x.x.x.x` IP should be secured and therefore uses ESP (Encapsulating Security Payload) protocol to the packet and adds a new IP header that points to the VPN gateway public IP address.

Before ESP can be used, the local computer would have to use IKE with the VPN server to come out with a secret key that will only both the client and the VPN would know and that key would be used by the OS to encrypt and decrypt the message.

## OpenVPN and TLS
OpenVPN relies on TLS but only for the control channel - the initial handshake to authenticate and swap encryption keys. OpenVPN defaults to a custom security protocol built over UDP, instead of TCP, to transmit data across the data channel.

Why not use TLS since the messages are already encrypted? The main benefit of using OpenVPN is that it hides the target server IP address over a private IP and sends the public IP of the VPN server. Therefore, we also hide the destination of our message.


## Conclusion
In short, OpenVPN 3 brings a modern, modular approach to Linux VPNs. Instead of one massive program, it uses separate background services to smoothly handle your connection, virtual network interfaces, and DNS routing. While alternatives like IPsec work deeper in the operating system, OpenVPN 3 offers an excellent balance of flexibility, strong encryption, and privacy. Ultimately, understanding these core concepts makes setting up, securing, and troubleshooting your network traffic much easier.


<div id="refs" class="references">
1. renatolfc (GitHub Gist). A sample OpenVPN client configuration file in the unified format.<a href="https://gist.github.com/renatolfc/f6c9e2a5bd6503005676" target="_blank" rel="noopener noreferrer">https://gist.github.com/renatolfc/f6c9e2a5bd6503005676</a>
<br>
 2. linuxrootroom (Medium). Why is /etc/resolv.conf Configured to Use 127.0.0.53 on Linux.<a href="https://medium.com/@linuxrootroom/dd-721dc2b25d1c" target="_blank" rel="noopener noreferrer">https://medium.com/@linuxrootroom/dd-721dc2b25d1c</a>
<br>
 3. ChatGPT. Shared Conversation. <a href="https://chatgpt.com/share/e/6999cc4f-531c-800b-9420-1256869fea3b" target="_blank" rel="noopener noreferrer">https://chatgpt.com/share/e/6999cc4f-531c-800b-9420-1256869fea3b</a>
</div>