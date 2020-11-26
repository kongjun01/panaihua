---
layout: post
title: "pyppeteer 内存泄漏排查"
date: 2020-11-13
excerpt: "收到告警后，笔者先登录到告警机器中，
 top 命令查看此时此刻的各个应用程序占用的内存大小，发现没有占用很大内存的进程。
 执行 `ps -ef` 发现有很多chromium,查了资料都说是僵尸进程，但是僵尸进程应该不占用内存和cpu的..."
categories: 
- 爬虫
tags:
- pyppeteer
comments: true
---

> 在工作中很少能够碰到内存泄漏的问题，但是一旦遇到了，就是一个比较难解的问题，
 本文旨在记录这次在问题排查的过程中，一些思路和排查方向
 
 收到告警后，笔者先登录到告警机器中，
 top 命令查看此时此刻的各个应用程序占用的内存大小，发现没有占用很大内存的进程。
 执行 `ps -ef` 发现有很多chromium,查了资料都说是僵尸进程，但是僵尸进程应该不占用内存和cpu的。
 
 > 僵尸子进程已经放弃了几乎所有的内存空间，没有任何可执行代码，也不能被调度，仅仅在进程列表中保留一个位置，记载该进程的退出状态信息供其他进程收集，除此之外，僵尸进程不再占有任何存储空间。他需要他的父进程来为他收尸，如果他的父进程没有安装SIGCHLD信号处理函数调用wait 或 waitpid() 等待子进程结束，也没有显式忽略该信号，那么它就一直保持僵尸状态，如果这时候父进程结束了，那么init进程会自动接手这个子进程，为他收尸，他还是能被清除掉的。但是如果父进程是一个循环，不会结束，那么子进程就会一直保持僵尸状态，这就是系统中为什么有时候会有很多的僵尸进程。

##### 第一步先内存分析
网上有很多方式，没几个好用的。后来发现一款工具 `pyrasite`, 可以直接连上一个正在运行的python程序，打开一个类似python的交互终端来运行命令、检查程序状态。很像阿里开源的的Arthas，这种真的超级方便。

首先安装

```
# pip install pyrasite
```

连接到有问题的python程序，开始收集信息

```
pyrasite-shell <pid>
>>>
```

接下来就可以在<pid>进程里调用任意python代码，查看进程状态。
所以就可以使用 guppy 获取内存使用的各种对象占用情况，可以打印各种对象所占空间大小，如果python进程中有未释放的对象，造成内存占用升高，可通过guppy查看。

```
# pip install guppy
from guppy import hpy
h = hpy()
h.heap()
```

结果运行如下：

```
# Partition of a set of 48477 objects. Total size = 3265516 bytes.
# Index Count % Size % Cumulative % Kind (class / dict of class)
# 0 25773 53 1612820 49 1612820 49 str
# 1 11699 24 483960 15 2096780 64 tuple
# 2 174 0 241584 7 2338364 72 dict of module
# 3 3478 7 222592 7 2560956 78 types.CodeType
# 4 3296 7 184576 6 2745532 84 function
# 5 401 1 175112 5 2920644 89 dict of class
# 6 108 0 81888 3 3002532 92 dict (no owner)
# 7 114 0 79632 2 3082164 94 dict of type
# 8 117 0 51336 2 3133500 96 type
# 9 667 1 24012 1 3157512 97 __builtin__.wrapper_descriptor
# <76 more rows. Type e.g. '_.more' to view.>
```

发现占用最大的是str，占比很小，所以确定就是僵尸进程的问题了。

##### 解决僵尸进程
使用专门的 init 进程 [查看文章](https://juejin.im/post/6844904029248552973)
原来的启动方式：

```
#ENTRYPOINT gunicorn -w 1 --threads 60 -b 0.0.0.0:5000 server:app
```

变更后：

```
ENTRYPOINT ["dumb-init", "--"]
CMD ["bash", "-c", "gunicorn -w 1 --threads 60 -b 0.0.0.0:5000 server:app"]
```

运行一段时间进程照样还是多，问题没有解决

##### 手动关闭chrome进程

[原文地址](https://stackoverflow.com/questions/53939503/puppeteer-doesnt-close-browser)

修改之后的代码：

```python
if browser is not None:
    await browser.close()
		os.system('pkill chrome')
```

问题还是没有解决

##### 从代码方面解决

通过调试程序，发现程序一直卡在这里，当没有这个节点的时候一直在等待

```
title_elements = await page.xpath('//*[@class="cc-button cc-button-default ad-login-comp-btn "]/div')
```

修改为问题解决

```
await page.waitForXPath('//*[@class="cc-button cc-button-default ad-login-comp-btn "]/div', {'timeout': 2000})
```




