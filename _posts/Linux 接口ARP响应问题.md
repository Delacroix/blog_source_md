---
title: Linux 接口ARP响应问题
date: 2016-12-22 14:48:33
categories:
- Linux
tags:
- Operations
- Network
- ARP
---
# Linux 接口ARP响应问题

# arp_announce/arp_ignore sysctl
The arp_announce/arp_ignore sysctl on interfaces is available at the Linux official kernel since 2.6.4 and 2.4.26. The description about arp_announce/arp_ignore taken from kernel documentation is as follows: 
## arp_announce - INTEGER
Define different restriction levels for announcing the local source IP address from IP packets in ARP requests sent on interface:

**0 - (default) Use any local address, configured on any interface**

**1 - Try to avoid local addresses that are not in the target's subnet for this interface.** 
This mode is useful when target hosts reachable via this interface require the source IP address in ARP requests to be part of their logical network configured on the receiving interface. When we generate the request we will check all our subnets that include the target IP and will preserve the source address if it is from such subnet. If there is no such subnet we select source address according to the rules for level 2.

**2 - Always use the best local address for this target.** 
In this mode we ignore the source address in the IP packet and try to select local address that we prefer for talks with the target host. Such local address is selected by looking for primary IP addresses on all our subnets on the outgoing interface that include the target IP address. If no suitable local address is found we select the first local address we have on the outgoing interface or on all other interfaces, with the hope we will receive reply for our request and even sometimes no matter the source IP address we announce. The max value from conf/{all,interface}/arp_announce is used. Increasing the restriction level gives more chance for receiving answer from the resolved target while decreasing the level announces more valid sender's information.

## arp_ignore - INTEGER
Define different modes for sending replies in response to received ARP requests that resolve local target IP addresses:
**0 - (default): reply for any local target IP address, configured on any interface**

**1 - reply only if the target IP address is local address configured on the incoming interface**

**2 - reply only if the target IP address is local address configured on the incoming interface and both with the sender's IP address are part from same subnet on this interface**

**3 - do not reply for local addresses configured with scope host, only resolutions for global and link addresses are replied**

**4-7 - reserved**

**8 - do not reply for all local addresses**
The max value from conf/{all,interface}/arp_ignore is used when ARP request is received on the {interface}

## Disable ARP for VIP
To disable ARP for VIP at real servers, we just need to set arp_announce/arp_ignore sysctls at the interface connected to the VIP network. 
For example, real servers have eth0 connected to the VIP network with the VIP at interface lo, we will have the following commands.

```
echo 1 > /proc/sys/net/ipv4/conf/eth0/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/eth0/arp_announce
```

Or, if /etc/sysctl.conf is used in the system, we have this config in /etc/sysctl.conf

```
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.eth0.arp_announce = 2
```

If you're about to brought up a new logical interface(eg. lo.0 etc.) it's recommended to have the following commands.

```
echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce
```

>**Note that the arp_announce/arp_ignore sysctls must be setup correctly, before the VIP address is brought up at a logical interface at real servers.**