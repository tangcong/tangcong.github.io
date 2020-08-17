---
layout: post
title: iptables how to work 
description: linux,iptables,netfilter 
date:     2019-01-12
author:     "唐聪"
categories: [ "Tech"]
tags:
    - iptables
---

## iptables介绍 

## iptables 初识  

### iptables 接口

### iptables table 

### iptables chain 

### iptables 



## iptables 内核实现之[netfilter](https://en.wikipedia.org/wiki/Netfilter)

### netfilter 介绍

netfilter is a set of hooks inside the Linux kernel that allows kernel modules to register callback functions with the network stack. A registered callback function is then called back for every packet that traverses the respective hook within the network stack.

netfilter设计实现上采用了unix的分离原则: 策略同机制分离, 接口同引擎分离.

#### Main Feature

* stateless packet filtering (IPv4 and IPv6)
* stateful packet filtering (IPv4 and IPv6)
* all kinds of network address and port translation, e.g. NAT/NAPT (IPv4 and IPv6)
* flexible and extensible infrastructure
* multiple layers of API's for 3rd party extensions.

源码分析，以linux内核版本以linux-3.16.62为例.

### ipv4 协议栈

### netfilter hook

```
arp.c:688:	NF_HOOK(NFPROTO_ARP, NF_ARP_OUT, skb, NULL, skb->dev, dev_queue_xmit);
arp.c:975:	return NF_HOOK(NFPROTO_ARP, NF_ARP_IN, skb, dev, NULL, arp_process);
ip_forward.c:141:	return NF_HOOK(NFPROTO_IPV4, NF_INET_FORWARD, skb, skb->dev,
ip_input.c:256:	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, skb, skb->dev, NULL,
ip_input.c:457:	return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,
ipmr.c:1785:	NF_HOOK(NFPROTO_IPV4, NF_INET_FORWARD, skb, skb->dev, dev,
ip_output.c:311:				NF_HOOK(NFPROTO_IPV4, NF_INET_POST_ROUTING,
ip_output.c:327:			NF_HOOK(NFPROTO_IPV4, NF_INET_POST_ROUTING, newskb,
ip_output.c:331:	return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL,
ip_output.c:345:	return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL, dev,
raw.c:410:	err = NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_OUT, skb, NULL,
xfrm4_input.c:64:	NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, skb->dev, NULL,
xfrm4_output.c:99:	return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb,
```

### [Packet Flow](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)

netfilter packet 数据流如下:

![netfilter flow](/img/netfilter-packet-flow.png)

## netfilter/iptables 应用

* build internet firewalls based on stateless and stateful packet filtering
* deploy highly available stateless and stateful firewall clusters
* use NAT and masquerading for sharing internet access if you don't have enough public IP addresses
* use NAT to implement transparent proxies
* aid the tc and iproute2 systems used to build sophisticated QoS and policy routers
* do further packet manipulation (mangling) like altering the TOS/DSCP/ECN bits of the IP header
