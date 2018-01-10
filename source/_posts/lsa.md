---
title: ospf lsa理解（转）
categories: ospf
tags:
  - ospf
---
OSPF的区域类型和LSA类型这两个知识点，经常容易导致人们对他们理解上的混乱，今天就来谈谈这两个概念。
本文将会以下面这个拓扑图为例进行讲解。

# LSA Type，链路状态通告的类型
本来想先讲区域类型，但是由于要理解不同类型的区域，必然要涉及到不同类型的LSA，现在把LSA放到前面先讲。
我们这里谈谈常见的几种LSA：1、2、3、4、5、7类LSA。

## 1类LSA，路由器LSA。
OSPF网络中所有路由器都会产生1类LSA，他的意思就是表示路由器自己在本区域内的直连链路信息。该LSA仅在本区域内传播。其中，Link ID跟ADV Router写的都是该路由器的RouterID。
下图为1类LSA：

## 2类LSA，网络LSA。
在广播或者非广播模式下（NBMA）由DR生成。该LSA仅在本区域内传播。2类LSA表达的意思应该是：某区域内，在广播或非广播的网段内选举了DR，于是DR在本区域范围利用2类LSA来进行通告。该LSA仅在本区域内传播。其中，该LSA的Link ID就是该DR的接口IP地址，而ADV Router则是DR的Router ID。

## 3类LSA，网络汇总LSA。
由区域边界路由器ABR生成，用于将一个区域内的网络通告给OSPF中的其他区域。可以认为3类LSA保存着本区域以外的所有其他区域的网络。举个例子，在多区域的环境如1-0-2这样的三个区域，含有area1和area0的ABR会把area1的网络以3类LSA的形式通告给area0，当然它也会把area0里面的网络通告给area1。那么，area1里面的网络又是如何通告到area2呢？这里就要考虑到area1那些一开始被转换成3类LSA的网络，是如何进入到area2的问题了。当原先这个3类LSA进入到area0跟area2的边界路由器时，位于这个边界的ABR就把这条包含着area1链路信息的3类LSA进行修改，修改的内容是把里面的ADV Router替换成自己的Router ID，并且维持原先的Link ID不变，然后把这条修改后的LSA通告给area2，这个就是3类LSA的工作过程。

## 4类LSA，ASBR汇总LSA。
4类LSA跟5类LSA是紧密联系在一起的，可以说4类LSA是由于5类LSA的存在而产生的。4类LSA由距离本路由器最近的ABR生成，这句话应该要这样来理解：如果路由器想要找到包含了外部路由的那台ASBR（自治系统边界路由器）的话，你应该要到达哪台ABR，这台ABR的Router ID就写在该LSA的ADV Router里面，而LSA里面的Link ID代表的是该ASBR的Router ID。

## 5类LSA，外部的LSA。
5类LSA由包含了外部路由的ASBR产生，目标是把某外部路由通告给OSPF进程的所有区域（特殊区域除外，下面会提到）。5类LSA可以穿越所有区域，意思是在跨区域通告时，该LSA的Link ID和ADV Router一直保持不变。通俗一点来说，就像是该ASBR对OSPF全网络的所有路由器说，我有这个外部路由，想去的话就来找我吧！其中，Link ID代表的是那台ASBR所引入的网络，ADV Router则是该ASBR的Router ID。
下图为5类LSA：

## 7类LSA
7类LSA是一种由NSSA区域中引入了外部路由的路由器生成的LSA，他仅在NSSA本区域内传播。由于NSSA区域不允许外部的路由进来从而禁止了5类LSA，那么为了能够把自己的外部路由传播出去，于是使用了7类LSA来代替5类LSA的功能。值得注意的一点是，当这种7类LSA到达NSSA跟其他区域的边界后，该边界路由器会根据这条7类LSA。生成对应的5类LSA然后继续传播给其他区域。此时，这条5类的LSA里面的Link ID跟7类LSA一样，都是该外部网络地址，而ADV Router则变成了该边界路由器的Router ID，因为这条5类LSA本来就是边界路由器产生的。这里要注意的一点是，该5类LSA里面的Forwarding Address还是保持跟之前的7类LSA的Forwarding Address一样。

# Area Type，区域类型
OSPF的区域类型分为5种：Backbone area(area 0)、Standard area、Stub area、Totally stubby area、No so stubby area(NSSA)。下面来逐一介绍。

## 1、Backbone area
也叫骨干区域，其实就是area 0。根据OSPF的设计原则，area 0在OSPF网络中起着中心节点的作用，其他区域的链路信息通过area 0来进行相互传递，这也意味着所有其他区域都必须跟area 0相连。该区域支持1、2、3、4、5类LSA。

## 2、Standard area
也叫标准区域，标准区域的意思就是在这个区域里面可以正常传递OSPF各类报文。该区域支持1、2、3、4、5类LSA。

## 3、Stub area
也叫末节区域，所谓末节区域，意思就是该区域不接受非OSPF网络的任何外部路由（external route），它如果要到达那些外部路由的时候，只需要通过默认路由把它发出去就可以了。该区域支持1、2、3类LSA。

## 4、Totally stubby area
也叫完全末节区域，他的意思是该区域非但不接受外部路由，也不接受自己本区域以外的其他区域的链路信息。它如果要到达本区域以外的目标网络的时候，也是跟末节区域一样，直接把报文通过默认路由发出去。这里要注意的是，由于默认路由是用3类LSA发送的，所以完全末节区域虽然不允许普通的3类LSA报文，但是支持这种包含默认路由的3类LSA。该区域支持1、2类LSA，以及包含默认路由的3类LSA。

## 5、No so stubby area
就是平时所说的NSSA了，这个NSSA其实是从stub区域发展而来的，它的意思是在含有stub区域的条件下，还拥有可以发送外部路由出去给其他区域的能力。该区域支持1、2、3、7类LSA。这里注意一点的是，NSSA区域还有另外一种模式，那就是是完全末节区域模式的NSSA。这个模式其实就是在完全末节区域环境下允许引入外部路由，这种区域模式支持1、2类LSA以及包含默认路由的3类LSA。

# 配置
```
area 24 stub
area 24 stub no-summary
```
