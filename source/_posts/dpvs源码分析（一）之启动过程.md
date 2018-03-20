---
title: dpvs源码分析（一）之启动过程
categories: 网络转发
tags:
  - 网络转发
  - DPVS
  - DPDK
---

本文用于分析dpvs的启动流程，会对主要逻辑进行解析，忽略了一些边缘代码，比如配置文件解析，函数指针的注册等等。在阅读主逻辑的时候，如果有疑问的地方，再去看一些配置相关，初始化相关的代码。这样不仅高效而且不会那么枯燥。被忽略的代码将在本文中用...代替。
# 从main函数开始
在src/mian.c文件中，main函数还是比较清晰的。首先是初始 -> 然后启动端口 -> 然后启动工作线程 -> 主线程进入循环。
{% codeblock lang:c %}
int main(int argc, char *argv[])
{
    ...
    //各种初始化，暂不关心，用到的时候再看。
    err = rte_eal_init(argc, argv);
    if (err < 0)
        rte_exit(EXIT_FAILURE, "Invalid EAL parameters\n");
    argc -= err, argv += err;

    rte_timer_subsystem_init();

    if ((err = cfgfile_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail init configuration file: %s\n",
                 dpvs_strerror(err));

    if ((err = netif_virtual_devices_add()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail add virtual devices:%s\n",
                 dpvs_strerror(err));

    if ((err = dpvs_timer_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail init timer on %s\n", dpvs_strerror(err));

    if ((err = tc_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init traffic control: %s\n",
                 dpvs_strerror(err));

    if ((err = netif_init(NULL)) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init netif: %s\n", dpvs_strerror(err));
    /* Default lcore conf and port conf are used and may be changed here 
     * with "netif_port_conf_update" and "netif_lcore_conf_set" */

    if ((err = ctrl_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init ctrl plane: %s\n",
                 dpvs_strerror(err));

    if ((err = tc_ctrl_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init tc control plane: %s\n",
                 dpvs_strerror(err));

    if ((err = vlan_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init vlan: %s\n", dpvs_strerror(err));

    if ((err = inet_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init inet: %s\n", dpvs_strerror(err));

    if ((err = sa_pool_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init sa_pool: %s\n", dpvs_strerror(err));

    if ((err = dp_vs_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init ipvs: %s\n", dpvs_strerror(err));

    if ((err = netif_ctrl_init()) != EDPVS_OK)
        rte_exit(EXIT_FAILURE, "Fail to init netif_ctrl: %s\n",
                 dpvs_strerror(err));

    /* config and start all available dpdk ports */
    nports = rte_eth_dev_count();
    for (pid = 0; pid < nports; pid++) {
        dev = netif_port_get(pid);
        if (!dev) {
            RTE_LOG(WARNING, DPVS, "port %d not found\n", pid);
            continue;
        }
        //启动端口，配置端口。比如端口，队列，cpu的绑定等等。
        err = netif_port_start(dev);
        if (err != EDPVS_OK)
            RTE_LOG(WARNING, DPVS, "Start %s failed, skipping ...\n",
                    dev->name);
    }

    /* print port-queue-lcore relation */
    netif_print_lcore_conf(pql_conf_buf, &pql_conf_buf_len, true, 0);
    RTE_LOG(INFO, DPVS, "\nport-queue-lcore relation array: \n%s\n",
            pql_conf_buf);

    /* start data plane threads */
    netif_lcore_start(); //这里就是干活的线程，通过跟踪这个函数，最后会调用到netif_loop，即工作线程的loop

    /* write pid file */
    if (!pidfile_write(DPVS_PIDFILE, getpid()))
        goto end;

    timer_sched_loop_interval = dpvs_timer_sched_interval_get();
    assert(timer_sched_loop_interval > 0);

    dpvs_state_set(DPVS_STATE_NORMAL);

    /* start control plane thread */
    // 主线程循环，用于处理ctrl plane等消息，这里占不讨论，后续文章再讨论。
    while (1) {
        /* reload configuations if reload flag is set */
        try_reload();
        /* IPC loop */
        sockopt_ctl(NULL);
        /* msg loop */
        msg_master_process();

        /* timer */
        loop_cnt++;
        if (loop_cnt % timer_sched_loop_interval == 0)
            rte_timer_manage();
        /* kni */
        kni_process_on_master();

        /* process mac ring on master */
        neigh_process_ring(NULL);
 
        /* increase loop counts */
        netif_update_master_loop_cnt();
    }

	...

    exit(0);
}
{% endcodeblock %}

# DPVS dataplane线程，即工作线程 
DPVS在netif.c文件中static int netif_loop(void * dummy) 函数中收取，处理和发送数据包。
{% codeblock lang:c %}
static int netif_loop(void *dummy)
{
	...

    try_isol_rxq_lcore_loop();
    if (0 == lcore_conf[lcore2index[cid]].nports) {
        RTE_LOG(INFO, NETIF, "[%s] Lcore %d has nothing to do.\n", __func__, cid);
        return EDPVS_IDLE;
    }
    /* 这里收包和处理CPU分离，是指收包的CPU和处理包的CPU不同，应该是为了增加网卡的吞吐能力吧。
    只有在/etc/dpvs.conf配置文件中配置了isol_rx_cpu_ids才会生效，暂时用不到，在后文中再来分析吧*/

    list_for_each_entry(job, &netif_lcore_jobs[NETIF_LCORE_JOB_INIT], list) {
        do_lcore_job(job);
    }
    /* NETIF_LCORE_JOB_INIT type类型暂时没有用到，忽略之*/

    while (1) {
#ifdef CONFIG_RECORD_BIG_LOOP
        loop_start = rte_get_timer_cycles();
#endif
/* CONFIG_RECORD_BIG_LOOP给统计和debug用，暂不关心 */

        lcore_stats[cid].lcore_loop++;
        list_for_each_entry(job, &netif_lcore_jobs[NETIF_LCORE_JOB_LOOP], list) {
            do_lcore_job(job);
        }
        ++netif_loop_tick[cid];
        list_for_each_entry(job, &netif_lcore_jobs[NETIF_LCORE_JOB_SLOW], list) {
            if (netif_loop_tick[cid] % job->skip_loops == 0) {
                do_lcore_job(job);
                //netif_loop_tick[cid] = 0;
            }
        }
        /* 上面代码有点类似于netfilter处理链，数据包会被处理链的job依次处理, 这些job链放在netif_lcore_jobs 这个全局变量中。
        那么，有哪些job呢，在什么地方初始化这个全局变量的呢？他们的处理顺序是怎么样的？在下一节会有说明*/
#ifdef CONFIG_RECORD_BIG_LOOP
        loop_end = rte_get_timer_cycles();
        loop_time = (loop_end - loop_start) * 1E6 / cycles_per_sec;
        if (loop_time > longest_lcore_loop[cid]) {
            RTE_LOG(WARNING, NETIF, "update longest_lcore_loop[%d] = %d (<- %d)\n",
                    cid, loop_time, longest_lcore_loop[cid]);
            longest_lcore_loop[cid] = loop_time;
        }
        if (loop_time > BIG_LOOP_THRESH) {
            print_job_time(buf, sizeof(buf));
            RTE_LOG(WARNING, NETIF, "lcore[%d] loop over %d usecs (actual=%d, max=%d):\n%s\n",
                    cid, BIG_LOOP_THRESH, loop_time, longest_lcore_loop[cid], buf);
        }
#endif
/* CONFIG_RECORD_BIG_LOOP给统计和debug用，暂不关心 */
    }
    return EDPVS_OK;
}

//那么do_lcore_job 做了什么呢？ 其实do_lcore_job就是调用了job结构体中注册的函数指针指向的函数。

static inline void do_lcore_job(struct netif_lcore_loop_job *job)
{
#ifdef CONFIG_RECORD_BIG_LOOP
    uint64_t job_start, job_end;
    job_start = rte_get_timer_cycles();
#endif

    job->func(job->data);
    // 这里就是真正干活的地方了。
    // func 函数指针在main函数中进行了初始化，初始化过程及代码位置在下一节会讲到。

#ifdef CONFIG_RECORD_BIG_LOOP
    job_end = rte_get_timer_cycles();
    job->job_time[rte_lcore_id()] = (job_end - job_start) * 1E6 / cycles_per_sec;
#endif
}
{% endcodeblock %}


# DPVS dataplane线程job初始化（ 也就是 netif_lcore_jobs 全局变量的初始化）
目前job的类型为，又有NETIF_LCORE_JOB_LOOP和NETIF_LCORE_JOB_SLOW有用到：
```
enum netif_lcore_job_type {
    NETIF_LCORE_JOB_INIT      = 0,
    NETIF_LCORE_JOB_LOOP      = 1,
    NETIF_LCORE_JOB_SLOW      = 2,
    NETIF_LCORE_JOB_TYPE_MAX  = 3,
};
```

## job注册函数
netif_lcore_loop_job_register 函数将job这测到netif_lcore_jobs这个全局变量中。
{% codeblock lang:c %}
int netif_lcore_loop_job_register(struct netif_lcore_loop_job *lcore_job)
{
    struct netif_lcore_loop_job *cur;
    if (unlikely(NULL == lcore_job))
        return EDPVS_INVAL;

    list_for_each_entry(cur, &netif_lcore_jobs[lcore_job->type], list) {
        if (cur == lcore_job) {
            return EDPVS_EXIST;
        }
    }

    if (unlikely(NETIF_LCORE_JOB_SLOW == lcore_job->type && lcore_job->skip_loops <= 0))
        return EDPVS_INVAL;

    list_add_tail(&lcore_job->list, &netif_lcore_jobs[lcore_job->type]);
    //netif_lcore_jobs 记录job的全局变量，这个在netif_loop用到了
    return EDPVS_OK;
}
{% endcodeblock %}

那么在哪些位置调用了netif_lcore_loop_job_register 注册job，通过阅读源代码，可以发现注册NETIF_LCORE_JOB_LOOP， NETIF_LCORE_JOB_SLOW 这两种类型的job分布在如下所示位置。
## NETIF_LCORE_JOB_LOOP job注册
** 第一处： main->netif_init->netif_lcore_init函数中：** 
{% codeblock lang:c %}
/* register lcore jobs*/
snprintf(netif_jobs[0].name, sizeof(netif_jobs[0].name) - 1, "%s", "recv_fwd");
netif_jobs[0].func = lcore_job_recv_fwd;
netif_jobs[0].data = NULL;
netif_jobs[0].type = NETIF_LCORE_JOB_LOOP;
snprintf(netif_jobs[1].name, sizeof(netif_jobs[1].name) - 1, "%s", "xmit");
netif_jobs[1].func = lcore_job_xmit;
netif_jobs[1].data = NULL;
netif_jobs[1].type = NETIF_LCORE_JOB_LOOP;
snprintf(netif_jobs[2].name, sizeof(netif_jobs[2].name) - 1, "%s", "timer_manage");
netif_jobs[2].func = lcore_job_timer_manage;
netif_jobs[2].data = NULL;
netif_jobs[2].type = NETIF_LCORE_JOB_LOOP;
{% endcodeblock %}

** 第二处： main->ctrl_init->msg_init**
{% codeblock lang:c %}
ctrl_lcore_job.func = slave_lcore_loop_func;
ctrl_lcore_job.data = NULL;
ctrl_lcore_job.type = NETIF_LCORE_JOB_LOOP;
if ((ret = netif_lcore_loop_job_register(&ctrl_lcore_job)) < 0) {
    RTE_LOG(ERR, MSGMGR, "%s: fail to register ctrl func on slave lcores\n", __func__);
    return ret;
}
{% endcodeblock %}

## NETIF_LCORE_JOB_SLOW job注册
** 第一处： main->inet_init -> ipv4_init-> ipv4_frag_init **
{% codeblock lang:c %}
frag_job.func = ipv4_frag_job;
frag_job.data = NULL;
frag_job.type = NETIF_LCORE_JOB_SLOW;
frag_job.skip_loops = IP4_FRAG_FREE_DEATH_ROW_INTERVAL;
err = netif_lcore_loop_job_register(&frag_job);
{% endcodeblock %}


** 第二处： mian->inet_init -> neigh_init -> arp_init **
{% codeblock lang:c %}
neigh_sync_job.func = neigh_process_ring;
neigh_sync_job.data = NULL;
neigh_sync_job.type = NETIF_LCORE_JOB_SLOW;
neigh_sync_job.skip_loops = NEIGH_PROCESS_MAC_RING_INTERVAL;
err = netif_lcore_loop_job_register(&neigh_sync_job);
{% endcodeblock %}


** 通过上述分析，那么我们可以知道job的处理流程为 **
以下流程虽然都要执行，但是函数前后并不是强制依赖，比如lcore_job_timer_manage 不依赖于lcore_job_xmit的执行结果
```
lcore_job_recv_fwd -> lcore_job_xmit -> lcore_job_timer_manage -> slave_lcore_loop_func ->
ipv4_frag_job -> neigh_process_ring

```
这些函数的具体功能，将在下一章节进行分析。

启动过程就到此结束了，若有疑问，欢迎发邮件和我联系。



