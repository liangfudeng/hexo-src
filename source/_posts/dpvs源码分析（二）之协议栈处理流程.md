---
title: dpvs源码分析（二）之概述协议栈处理流程
categories: 网络转发
tags:
  - 网络转发
  - DPVS
  - DPDK
---

本文仅仅概述dpvs二三层协议栈的处理流程，并不对某个函数，某块进行详细分析，以避免刚刚接触DPVS的同学被这些细节扰乱视听。后面的章节将会针对于某个某块进行重点分析。废话不多说，开始吧。


{% codeblock lang:c %}
static struct pkt_type ip4_pkt_type = {
    //.type       = rte_cpu_to_be_16(ETHER_TYPE_IPv4),
    .func       = ipv4_rcv,
    .port       = NULL,
};
ip4_pkt_type.type = htons(ETHER_TYPE_IPv4);

static struct pkt_type arp_pkt_type = {
    //.type       = rte_cpu_to_be_16(ETHER_TYPE_ARP),
    .func       = neigh_resolve_input,
    .port       = NULL,
};
arp_pkt_type.type = rte_cpu_to_be_16(ETHER_TYPE_ARP);

{% endcodeblock %}

以上type都会被netif_register_pkt函数注册到全局变量pkt_type_tab中，提供给协议栈使用。目前只有ETHER_TYPE_ARP和ETHER_TYPE_IPv4被注册，也就是说dpvs协议栈目前仅仅会对这两种协议进行处理。其余的协议会通过kni传递个linux kernel。