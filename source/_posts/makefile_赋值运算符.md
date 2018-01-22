---
title: makefile中= := ?= +=
categories: 备忘录
tags:
  - makefile
---

Makefile 中:= ?= += =的区别
在Makefile中我们经常看到 = := ?= +=这几个赋值运算符，那么他们有什么区别呢？我们来做个简单的实验

新建一个Makefile，内容为：

```
ifdef DEFINE_VRE
    VRE = “Hello World!”
else
endif

ifeq ($(OPT),define)
    VRE ?= “Hello World! First!”
endif

ifeq ($(OPT),add)
    VRE += “Kelly!”
endif

ifeq ($(OPT),recover)
    VRE := “Hello World! Again!”
endif

all:
    @echo $(VRE)
```

敲入以下make命令：
```
make DEFINE_VRE=true OPT=define 输出：Hello World!
make DEFINE_VRE=true OPT=add 输出：Hello World! Kelly!
make DEFINE_VRE=true OPT=recover  输出：Hello World! Again!
make DEFINE_VRE= OPT=define 输出：Hello World! First!
make DEFINE_VRE= OPT=add 输出：Kelly!
make DEFINE_VRE= OPT=recover 输出：Hello World! Again!
```
从上面的结果中我们可以清楚的看到他们的区别了
= 是最基本的赋值
:= 是覆盖之前的值
?= 是如果没有被赋值过就赋予等号后面的值
+= 是添加等号后面的值

