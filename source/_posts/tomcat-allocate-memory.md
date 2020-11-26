---
layout: post
title: "Tomcat Cannot Allocate Memory"
date: 2016-04-29
excerpt: "今天测试环境的服务总是挂掉，上面部署了6个服务，也不怎么吃内存的。但是我这个服务总是启动之后过不了多久又挂了。"
categories: 
- Java
tags: 
- Memory
comments: true
---

今天测试环境的服务总是挂掉，上面部署了6个服务，也不怎么吃内存的。但是我这个服务总是启动之后过不了多久又挂了。tomcat日志如下：

````vim
Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00000007ca580000, 211288064, 0) failed; error='Cannot allocate memory' (errno=12)
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (malloc) failed to allocate 211288064 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /root/hs_err_pid5095.log
````

解决如下：

```shell
test01:~ root$ grep -i commit /proc/meminfo
CommitLimit:    20389524 kB
Committed_AS:   18541832 kB
```
说明：
>*CommitLimit*：最大能分配的内存(个人理解仅仅在vm.overcommit\_memory=2时候生效)，具体的值是SWAP内存大小 + 物理内存 * overcommit_ratio / 100  
*Committed\_AS*：当前已经分配的内存大小  
*overcommit\_memory*：规定决定是否接受超大内存请求的条件。  
>>这个参数有三个可能的值：  
0 — 默认设置。个人理解：当应用进程尝试申请内存时，内核会做一个检测。内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。举个例子，比如1G的机器，A进程已经使用了500M，当有另外进程尝试malloc 500M的内存时，内核就会进行check，发现超出剩余可用内存，就会提示失败。  
1 — 对于内存的申请请求，内核不会做任何check，直到物理内存用完，触发OOM杀用户态进程。同样是上面的例子，1G的机器，A进程500M，B进程尝试malloc 500M，会成功，但是一旦kernel发现内存使用率接近1个G(内核有策略)，就触发OOM，杀掉一些用户态的进程(有策略的杀)。  
2 — 当 请求申请的内存 >= SWAP内存大小 + 物理内存 * N，则拒绝此次内存申请。解释下这个N：N是一个百分比，根据overcommit\_ratio/100来确定，比如overcommit\_ratio=50，那么N就是50%。注意：只为swap区域大于其物理内存的系统推荐这个设置overcommit\_ratio将 overcommit_memory 设定为 2 时，指定所考虑的物理 RAM 比例。默认为 50。  

尝试修改内核参数如下：

````shell
test01:~ root$ vim /etc/sysctl.conf
vm.overcommit_memory = 1
````
运行使之生效:

````shell
test01:~ root$ sysctl -p
test01:~ root$ sysctl -n vm.overcommit_memory
1
````

上面的方法尝试过后还是没有用，内存还是不够。
最后找了一个释放内存的命令：

````shell
test01:~ root$ echo 1 > /proc/sys/vm/drop_caches
````
用free -m 查看内存压缩了一个g，待看看是否有效果吧～












