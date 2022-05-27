---
layout: post
title: "记一次 Binlog 的应用"
date: 2019-01-18
excerpt: "我们现在遇到一个问题，每次在发券系统里面增加一个表操作的时候，都会实现缓存，实现了缓存就要实现通知刷新，所以程序里只要有更新的操作都要硬编码发送一次消息。这个过程维护成本太大，就想到是否有什么方法可以实现自动通知刷新。"
categories: 
- Mysql
tags: 
- Binlog
comments: true
---

### 前言
我们现在遇到一个问题，每次在发券系统里面增加一个表操作的时候，都会实现缓存，实现了缓存就要实现通知刷新，所以程序里只要有更新的操作都要硬编码发送一次消息。这个过程维护成本太大，就想到是否有什么方法可以实现自动通知刷新。

所以我就在这里小小去了解一下 Binlog。

### 什么是Binlog
`MySQL` 有四种类型的日志——Error Log、General Query Log、Binary Log 和 Slow Query Log。

第一个是错误日志，记录 `MySQL` 的一些错误。第二个是一般查询日志，记录 `MySQL` 正在做的事情，比如客户端的连接和断开、来自客户端每条 `Sql Statement` 记录信息；如果你想准确知道客户端到底传了什么瞎 [哔哔] 玩意儿给服务端，这个日志就非常管用了，不过它非常影响性能。第四个是慢查询日志，记录一些查询比较慢的 SQL 语句——这种日志非常常用，主要是给开发者调优用的。

剩下的第三种就是`Binlog`了，包含了一些事件，这些事件描述了数据库的改动，如建表、数据改动等，也包括一些潜在改动，比如

```sql 
DELETE FROM xxx WHERE bing = xxx
```

然而一条数据都没被删掉的这种情况。除非使用 Row-based logging，否则会包含所有改动数据的`SQL Statement`。

显然，我们执行 `SELECT` 等不设计数据变更的语句是不会记录 `Binlog` 的，而涉及到数据更新则会记录。要注意的是，对支持事务的引擎如InnoDB而言，必须要提交了事务才会记录 `Binlog`。`Binlog` 是在事务最终commit前写入的，`Binlog` 什么时候刷新到磁盘跟参数sync_binlog相关。如果设置为0，则表示`MySQL`不控制 `Binlog` 的刷新，由文件系统去控制它缓存的刷新，而如果设置为不为0的值则表示每sync_binlog次事务，MySQL调用文件系统的刷新操作刷新 `Binlog` 到磁盘中。设为1是最安全的，在系统故障时最多丢失一个事务的更新，但是会对性能有所影响，一般情况下会设置为100或者0，牺牲一定的一致性来获取更好的性能.


### 流程设计

![](/images/binlog-1.png)

mysql-binlog-connector-server实现监听binlog，收到之后放入队列中，另起一个线程消耗队列。
利用钩子在宕机之前保存消费到的position。

>保存position的信息本来想是放在mysql，但是后期会增加HA，需要使用Zookeeper保持心跳，为了方便还是选择保存在Zookeeper

![](/images/binlog-2.png)

- EventListener: 在向mysql发送dump命令之前会先从Zookeeper中获取上次解析成功的 `Position` （如果是第一次启动，则获取初始指定位置或者当前数据段binlog位点）。mysql接受到dump命令后，由EventListener从mysql上pull binlog数据进行解析并传递给 `parse` (传递给parse 模块进行数据存储，是一个阻塞操作，直到存储成功)，传送成功之后更新 `Position`。

- MessagContaner: 保存解析成功的数据，交给 `DataSchdule` 处理。设计这个模块是为了缓冲，防止重复发送消息。

下面是模拟mysql的slave时收到的消息
> {before=[797, 3528, zjy-客户1-广告1, 100004, 2017-03-24, 2017-04-09, 1488857138210, 1488857138210, 0, 0, 2, 4, null, null, null, null, null, null, null, null, 1, null, null, null, null, null, null, Mon Mar 06 23:00:36 CST 2017, Thu Oct 12 03:23:18 CST 2017, 0, 0, 0, 26526708857909, null, 1, 0, 1.000], after=[797, 3528, zjy-客户1-广告1, 100004, 2017-10-11, 2017-04-09, 1488857138210, 1488857138210, 0, 0, 2, 4, null, null, null, null, null, null, null, null, 1, null, null, null, null, null, null, Mon Mar 06 23:00:36 CST 2017, Thu Oct 12 22:01:15 CST 2017, 0, 0, 0, 26526708857909, null, 1, 0, 1.000]},
    {before=[885, 3528, zjy-客户1-广告1, 333001, 2017-03-08, 2017-04-27, 1489993531824, 1489993531824, 0, 0, 2, 4, null, null, null, null, null, null, null, null, 1, null, null, null, null, null, null, Mon Mar 20 23:05:31 CST 2017, Thu Oct 12 01:45:16 CST 2017, 0, 0, 0, 27795375556109, null, 0, 0, 1.000], after=[885, 3528, zjy-客户1-广告1, 333001, 2017-10-11, 2017-04-27, 1489993531824, 1489993531824, 0, 0, 2, 4, null, null, null, null, null, null, null, null, 1, null, null, null, null, null, null, Mon Mar 20 23:05:31 CST 2017, Thu Oct 12 22:01:15 CST 2017, 0, 0, 0, 27795375556109, null, 0, 0, 1.000]},
    {before=[4882, 3531, zjy-客户2-广告1, 12200, 2017-04-14, 2017-04-14, 1492148740122, 1492148740122, 1, 0, 0, 3, null, null, null, null, null, null, null, null, 1, null, null, null, null, null, null, Fri Apr 14 21:44:11 CST 2017, Sat Sep 02 00:17:55 CST 2017, 0, 0, 0, 30055174218659, null, 0, 0, 1.000], after=[4882, 3531, zjy-客户2-广告1, 12200, 2017-10-11, 2017-04-14, 1492148740122, 1492148740122, 1, 0, 0, 3, null, null, null, null, null, null, null, null, 1, null, null, null, null, null, null, Fri Apr 14 21:44:11 CST 2017, Thu Oct 12 22:01:15 CST 2017, 0, 0, 0, 30055174218659, null, 0, 0, 1.000]}
]}}Event{header=EventHeaderV4{timestamp=1507788075000, eventType=XID, serverId=1, headerLength=19, dataLength=12, nextPosition=81793595, flags=0}, data=XidEventData{xid=3329634}}


### 总结

这里相当于一个迷你的 `Canal` ，还有很多场景没有实现，比如一致性，HA。如果有机会在参考 `Canal` 的设计。


参考资料：

[canal系列—Canal 的介绍](https://blog.csdn.net/u012758088/article/details/78788523)





