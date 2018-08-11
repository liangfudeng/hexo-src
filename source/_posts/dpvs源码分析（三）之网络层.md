---
title: dpvs源码分析（三）之网络层
categories: 原创精选
tags:
  - 网络转发
  - DPVS
  - DPDK
---

# ipv4报文处理

接**{% post_link dpvs源码分析（二）之链路层 %}** 我们知道在处理ipv4报文在下面的函数当中。
{% codeblock lang:c %}
static int ipv4_rcv(struct rte_mbuf *mbuf, struct netif_port *port)
{
    ...
      
	/*这个函数应该是参考了linux协议栈的处理。大概的意思是：
	 *pskb_may_pull确保skb->data指向的内存包含的数据至少为IP头部大小，由于每个
	 *IP数据包包括IP分片必须包含一个完整的IP头部。如果小于IP头部大小，则缺失
	 *的部分将从数据分片中拷贝。这些分片保存在skb_shinfo(skb)->frags[]中。
	 */
     //参考http://blog.chinaunix.net/uid-22577711-id-3220103.html
    if (mbuf_may_pull(mbuf, sizeof(struct ipv4_hdr)) != 0)
        goto inhdr_error;

    iph = ip4_hdr(mbuf);

    hlen = ip4_hdrlen(mbuf);
    if (((iph->version_ihl) >> 4) != 4 || hlen < sizeof(struct ipv4_hdr))
        goto inhdr_error;

    if (mbuf_may_pull(mbuf, hlen) != 0)
        goto inhdr_error;

    if (unlikely(!(port->flag & NETIF_PORT_FLAG_RX_IP_CSUM_OFFLOAD))) {
        if (unlikely(rte_raw_cksum(iph, hlen) != 0xFFFF))
            goto csum_error;
    }

    len = ntohs(iph->total_length);
    if (mbuf->pkt_len < len) {
        IP4_INC_STATS(intruncatedpkts);
        goto drop;
    } else if (len < hlen)
        goto inhdr_error;

    /* trim padding if needed */
    if (mbuf->pkt_len > len) {
        if (rte_pktmbuf_trim(mbuf, mbuf->pkt_len - len) != 0) {
            IP4_INC_STATS(indiscards);
            goto drop;
        }
    }
    mbuf->userdata = NULL;
    mbuf->l3_len = hlen;

#ifdef CONFIG_DPVS_IPV4_DEBUG
    ip4_dump_hdr(iph, mbuf->port);
#endif

    return INET_HOOK(INET_HOOK_PRE_ROUTING, mbuf, port, NULL, ipv4_rcv_fin);

csum_error:
    IP4_INC_STATS(csumerrors);
inhdr_error:
    IP4_INC_STATS(inhdrerrors);
drop:
    rte_pktmbuf_free(mbuf);
    return EDPVS_INVPKT;
}


int INET_HOOK(unsigned int hook, struct rte_mbuf *mbuf,
        struct netif_port *in, struct netif_port *out,
        int (*okfn)(struct rte_mbuf *mbuf))
{
    ...
    //执行hook函数
    ops = list_entry(hook_list, struct inet_hook_ops, list);

    if (!list_empty(hook_list)) {
        verdict = INET_ACCEPT;
        list_for_each_entry_continue(ops, hook_list, list) {
repeat:
            verdict = ops->hook(ops->priv, mbuf, &state);
            if (verdict != INET_ACCEPT) {
                if (verdict == INET_REPEAT)
                    goto repeat;
                break;
            }
        }
    }
    ...
    if (verdict == INET_ACCEPT || verdict == INET_STOP) {
        return okfn(mbuf); 
    } else if (verdict == INET_DROP) {
        rte_pktmbuf_free(mbuf); //继续协议栈处理
        return EDPVS_DROP;
    } else { /* INET_STOLEN */
        return EDPVS_OK; //交由下一个函数处理，
    }
}
{% endcodeblock %}
