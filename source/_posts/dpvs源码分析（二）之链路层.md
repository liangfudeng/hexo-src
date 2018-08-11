---
title: dpvs源码分析（二）之链路层
categories: 原创精选
tags:
  - 网络转发
  - DPVS
  - DPDK
---

本文仅仅概述dpvs二三层协议栈的处理流程，只会对重点函数和流程分析，以避免刚刚接触DPVS的同学被这些细节扰乱视听。后面的章节将会针对于某个某块进行重点分析。

接**{% post_link dpvs源码分析（一）之启动过程 %}**，我们知道，lcore_job_recv_fwd是首先被调动的函数：
{% codeblock lang:c %}
static void lcore_job_recv_fwd(void *arg)
{
    int i, j;
    portid_t pid;
    lcoreid_t cid;
    struct netif_queue_conf *qconf;

    cid = rte_lcore_id();
    assert(LCORE_ID_ANY != cid);

    for (i = 0; i < lcore_conf[lcore2index[cid]].nports; i++) {
        pid = lcore_conf[lcore2index[cid]].pqs[i].id;
        assert(pid < rte_eth_dev_count());

        for (j = 0; j < lcore_conf[lcore2index[cid]].pqs[i].nrxq; j++) {
            qconf = &lcore_conf[lcore2index[cid]].pqs[i].rxqs[j];
            
            // 从arp_ring获取arp报文，最后调用lcore_process_packets 处理，所以直接看lcore_process_packets这个函数就好了。
            lcore_process_arp_ring(qconf,cid);

            // 从网卡收包，存放于qconf->mbufs 结构体重，len为包的数量
            qconf->len = netif_rx_burst(pid, qconf);
            //统计
            lcore_stats_burst(&lcore_stats[cid], qconf->len);
            
            //处理数据报文，
            //mbuf会在lcore_process_packets被释放
            lcore_process_packets(qconf, qconf->mbufs, cid, qconf->len, 1);
            
            //将报文发送给Linux kernel
            kni_send2kern_loop(pid, qconf);
        }
    }
}
{% endcodeblock %}

{% codeblock lang:c %}
static void lcore_process_packets(struct netif_queue_conf *qconf, struct rte_mbuf **mbufs,
                      lcoreid_t cid, uint16_t count, bool pretetch)
{
    ...

    /* prefetch packets 
    预先将数据包从内存加载到cache，这样有可能加快运行速度*/
    if (pretetch) {
        for (t = 0; t < qconf->len && t < NETIF_PKT_PREFETCH_OFFSET; t++)
            rte_prefetch0(rte_pktmbuf_mtod(qconf->mbufs[t], void *));
    }

    /* L2 filter */
    for (i = 0; i < count; i++) {
        ...
        /*校验mac地址，如果和物理设备的mac地址一样，被设置为RTE_TYPE_HOST。不一样则被设置为ETH_PKT_OTHERHOST*/
        /* reuse mbuf.packet_type, it was RTE_PTYPE_XXX */
        mbuf->packet_type = eth_type_parse(eth_hdr, dev);

        /*
         * 如果通过dpip命令设置了设备forward2kni on，那么所有的报文都会复制一份给kernel
         * 所有数据包复制一份通过kni发送给kernel， 原有mbuf不变。
         */
        if (dev->flag & NETIF_PORT_FLAG_FORWARD2KNI) {
            if (likely(NULL != (mbuf_copied = mbuf_copy(mbuf,
                                pktmbuf_pool[dev->socket]))))
                kni_ingress(mbuf_copied, dev, qconf);
            else
                RTE_LOG(WARNING, NETIF, "%s: Failed to copy mbuf\n",
                        __func__);
        }

        /*
         * handle VLAN
         * if HW offload vlan strip, it's still need vlan module
         * to act as VLAN filter.
         * vlan_rcv会通过vlanid找到对应的dev，然后将dev id复制给mbuf->port_id
         */
        if (eth_hdr->ether_type == htons(ETH_P_8021Q) ||
            mbuf->ol_flags & PKT_RX_VLAN_STRIPPED) {
            if (vlan_rcv(mbuf, netif_port_get(mbuf->port)) != EDPVS_OK) {
                rte_pktmbuf_free(mbuf);
                lcore_stats[cid].dropped++;
                continue;
            }
            /*通过port id找到对应的dev设备*/
            dev = netif_port_get(mbuf->port);
            if (unlikely(!dev)) {
                rte_pktmbuf_free(mbuf);
                lcore_stats[cid].dropped++;
                continue;
            }
            /*获取二层头*/
            eth_hdr = rte_pktmbuf_mtod(mbuf, struct ether_hdr *);
        }
        /* handler should free mbuf */
        netif_deliver_mbuf(mbuf, eth_hdr->ether_type, dev, qconf,
                           (dev->flag & NETIF_PORT_FLAG_FORWARD2KNI) ? true:false,
                           cid, pkts_from_ring);
        ...
    }
}

/*上文用到了vlan_rcv这里对vlan_rcv做解析*/
int vlan_rcv(struct rte_mbuf *mbuf, struct netif_port *real_dev)
{
    ...
    // 剥离VLAN tag
    err = vlan_untag_mbuf(mbuf);
    if (unlikely(err != EDPVS_OK))
        return err;
    // 依据VLAN tag找到对应的VLAN设备
    dev = vlan_find_dev(real_dev, htons(ETH_P_8021Q),
                        mbuf_vlan_tag_get_id(mbuf));
    mbuf->port = dev->id;
    if (unlikely(mbuf->packet_type == ETH_PKT_OTHERHOST)) {
		/*这里通过目的地址判断，包是不是发送给vlan的。*/
        if (eth_addr_equal(&ehdr->d_addr, &dev->addr))
            mbuf->packet_type = ETH_PKT_HOST/*如果是ETH_PKT_OTHERHOST报文会被丢弃*/;
    }
    ...
}
{% endcodeblock %}

前面对vlan神马的都处理了，接下来就是二层报文的处理了：
{% codeblock lang:c %}
static inline int netif_deliver_mbuf(struct rte_mbuf *mbuf,
                                     uint16_t eth_type,
                                     struct netif_port *dev,
                                     struct netif_queue_conf *qconf,
                                     bool forward2kni,
                                     lcoreid_t cid,
                                     bool pkts_from_ring)
{
    ...
    pkt_type注册见下文，这里通过eth_type获取ptk_type
    其他协议，都会送给linux kernel
    pt = pkt_type_get(eth_type, dev);
    if (!forward2kni && NULL == pt) {
    	/*通过kni，发送给linux kernel*/
        kni_ingress(mbuf, dev, qconf);
        return EDPVS_OK;
    }
    ...

    /*clone arp pkt to every queue*/
    if (pt->type == rte_cpu_to_be_16(ETHER_TYPE_ARP) && !pkts_from_ring/*arp_ring里的报文，肯定不能再入ring了。*/) {
       /*将arp报文clone到每个队列，每个core维护自己的arp表现*/
       ...
    }

   /*Remove len bytes at the beginning of an mbuf. 移除二层头*/
    if (unlikely(NULL == rte_pktmbuf_adj(mbuf, sizeof(struct ether_hdr))))
        return EDPVS_INVPKT;

    /*在这里就开始处理上层协议了，目前只会处理ip和arp，也只注册了这两种*/
    err = pt->func(mbuf, dev);

    if (err == EDPVS_KNICONTINUE) {
        if (pkts_from_ring) {
        	/*pkt_from_ring为arp_ring过来的报文
        	* arp_ring过来的报文不再发给linux kernel，因此需要free mubf
        	*/
            rte_pktmbuf_free(mbuf);
            return EDPVS_OK;
        }

        if (!forward2kni && likely(NULL != rte_pktmbuf_prepend(mbuf,  (mbuf->data_off - data_off))))
        	// 发送给linux kernel
            kni_ingress(mbuf, dev, qconf);
    }

    return EDPVS_OK;
}
{% endcodeblock %}

不同类型的报文，注册不同的处理函数。
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
目前只有ETHER_TYPE_ARP和ETHER_TYPE_IPv4被注册，也就是说dpvs协议栈目前仅仅会对这两种协议进行处理。其余的协议会通过kni传递个linux kernel。
