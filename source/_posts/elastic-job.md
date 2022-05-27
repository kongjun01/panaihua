---
layout: post
title: "elastic-job 在广告系统中的实践总结"
date: 2017-12-26
excerpt: "一般来说，系统可使用消息传递代替部分使用作业的场景。两者确有相似之处。可互相替换的场景，如队列表。将待处理的数据放入队列表，然后使用频率极短的定时任务拉取队列表的数据并处理。这种情况使用消息中间件的推送模式可更好的处理实时性数据。"
categories: 
- Java
tags: 
- elastic-job
comments: true
---

### 为什么要使用elastic-job

##### 为什么要使用定时任务
一般来说，系统可使用消息传递代替部分使用作业的场景。两者确有相似之处。可互相替换的场景，如队列表。将待处理的数据放入队列表，然后使用频率极短的定时任务拉取队列表的数据并处理。这种情况使用消息中间件的推送模式可更好的处理实时性数据。而且基于数据库的消息存储吞吐量远远小于基于文件的顺序追加消息存储，但在某些场景下则不能互换：

* 时间驱动 OR 事件驱动：内部系统一般可以通过事件来驱动，但涉及到外部系统，则只能使用时间驱动。如：抓取外部系统价格。每小时抓取，由于是外部系统，不能像内部系统一样发送事件触发事件。

* 批量处理 OR 逐条处理：批量处理堆积的数据更加高效，在不需要实时性的情况下比消息中间件更有优势。而且有的业务逻辑只能批量处理，如：电商公司与快递公司结算，一个月结算一次，并且根据送货的数量有提成。比如，当月送货超过1000则额外给快递公司多1%的快递费。

* 非实时性 OR 实时性：虽然消息中间件可以做到实时处理数据，但有的情况并不需要。如：VIP用户降级，如果超过1年无购买行为，则自动降级。这类需求没有强烈的时间要求，不需要按照时间精确的降级VIP用户。

* 系统内部 OR 系统解耦：作业一般封装在系统内部，而消息中间件可用于系统间解耦。

##### 为什么要选择elastic-job

先比较下常见作业系统的差异：

* Quartz：Java事实上的定时任务标准。但Quartz关注点在于定时任务而非数据，并无一套根据数据处理而定制化的流程。虽然Quartz可以基于数据库实现作业的高可用，但缺少分布式并行执行作业的功能。
 
* TBSchedule：阿里早期开源的分布式任务调度系统。代码略陈旧，使用timer而非线程池执行任务调度。众所周知，timer在处理异常状况时是有缺陷的。而且TBSchedule作业类型较为单一，只能是获取/处理数据一种模式。还有就是文档缺失比较严重。

* Crontab：Linux系统级的定时任务执行器。缺乏分布式和集中管理功能。

> 之前广告系统使用的是Crontab，虽然也可以很好的支撑我们的需求，但是后来跑任务的这台机器有问题需要下线，然而运维迁移的时候任务没有迁移，导致所有的定时任务在那一晚全部失效。这是为什么我们需要迁移到elastic-job的主要原因

综上所述，当前存在的作业系统缺少分布式、并行调度、弹性扩容缩容、集中管理、定制化流程型任务等功能，所以需要一个新的作业系统完善这些功能。


### elastic-job介绍

elastic-job主要的设计理念是无中心化的分布式定时调度框架，思路来源于Quartz的基于数据库的高可用方案。但数据库没有分布式协调功能，所以在高可用方案的基础上增加了弹性扩容和数据分片的思路，以便于更大限度的利用分布式服务器的资源。

![](/images/struct.png)

##### 主要功能
* 分布式
	* 重写Quartz基于数据库的分布式功能，改用Zookeeper实现注册中心
* 并行调度
	* 采用任务分片方式实现。将一个任务拆分为n个独立的任务项，由分布式的服务器并行执行各自分配到的分片项
* 弹性扩容缩容
	* 将任务拆分为n个任务项后，各个服务器分别执行各自分配到的任务项。一旦有新的服务器加入集群，或现有服务器下线，elastic-job将在保留本次任务执行不变的情况下，下次任务开始前触发任务重分片
* 集中管理
	* 采用基于Zookeeper的注册中心，集中管理和协调分布式作业的状态，分配和监听。外部系统可直接根据Zookeeper的数据管理和- 监控elastic-job
* 定制化流程型任务
	* 作业可分为简单和数据流处理两种模式，数据流又分为高吞吐处理模式和顺序性处理模式，其中高吞吐处理模式可以开启足够多的线程快速的处理数据，而顺序性处理模式将每个分片项分配到一个独立线程，用于保证同一分片的顺序性
* 失效转移
	* 弹性扩容缩容在下次作业运行前重分片，但本次作业执行的过程中，下线的服务器所分配的作业将不会重新被分配。失效转移功能可以在本次作业运行中用空闲服务器抓取孤儿作业分片执行。同样失效转移功能也会牺牲部分性能
* 运行时状态收集
	* 监控作业运行时状态，统计最近一段时间处理的数据成功和失败数量，记录作业上次运行开始时间，结束时间和下次运行时间
* 作业停止，恢复和禁用
	* 用于操作作业启停，并可以禁止某作业运行（上线时常用）
* Spring命名空间支持
	* elastic-job可以不依赖于spring直接运行，但是也提供了自定义的命名空间方便与spring集成
* 运维平台
	* 提供web控制台用于管理作业和注册中心
* 稳定性
	* 在服务器无波动的情况下，并不会重新分片；即使服务器有波动，下次分片的结果也会根据服务器IP和作业名称哈希值算出稳定的分片顺序，尽量不做大的变动
* 高性能
	* 同一服务器的批量数据处理采用自动切割并多线程并行处理
* 灵活性
	* 所有在功能和性能之间的权衡，都可通过配置开启/关闭。如：elastic-job会将作业运行状态的必要信息更新到注册中心。如果作业执行频度很高，会造成大量Zookeeper写操作，而分布式Zookeeper同步数据可能引起网络风暴。因此为了考虑性能问题，可以牺牲一些功能，而换取性能的提升
* 幂等性
	* elastic-job可牺牲部分性能用以保证同一分片项不会同时在两个服务器上运行
* 容错性
	* 作业服务器与Zookeeper服务器通信失败则立即停止作业运行，防止作业注册中心将失效的分片分项配给其他作业服务器，而当前作业服务器仍在执行任务，导致重复执行

##### 流程图
作业启动流程图

![](/images/start.jpg)

* (1)第一台服务器上线触发主服务器选举。主服务器一旦下线，则重新触发选举，选举过程中阻塞，只有主服务器选举完成，才会执行其他任务。
* (2)某作业服务器上线时会自动将服务器信息注册到注册中心，下线时会自动更新服务器状态。
* (3)主节点选举，服务器上下线，分片总数变更均更新重新分片标记。
* (4)定时任务触发时，如需重新分片，则通过主服务器分片，分片过程中阻塞，分片结束后才可执行任务。如分片过程中主服务器下线，则先选举主服务器，再分片。
* (5)通过(4)可知，为了维持作业运行时的稳定性，运行过程中只会标记分片状态，不会重新分片。分片仅可能发生在下次任务触发前。
* (6)每次分片都会按服务器IP排序，保证分片结果不会产生较大波动。
* (7)实现失效转移功能，在某台服务器执行完毕后主动抓取未分配的分片，并且在某台服务器下线后主动寻找可用的服务器执行任务。	

### 使用中遇到的坑
首先看下最初的配置

```xml
<job:simple id="UpdateAdvertMatchTagJob" class="cn.com.duiba.tuia.core.biz.job.UpdateAdvertMatchTagJob" description="更新标签广告redis缓存数据" registry-center-ref="regCenter" cron="0 0/5 * * * ?"  sharding-total-count="1" event-trace-rdb-data-source="dataSource" />
```                
上面配置使用中遇到一个问题，就是第一次启动项目之后再修改这里的参数怎么都不生效。翻阅[官方文档](http://elasticjob.io/docs/elastic-job-lite/02-guide/config-manual/)发现有个overwrite的参数,它的意思是：
> 本地配置是否可覆盖注册中心配置，如果可覆盖，每次启动作业都以本地配置为准

通过这个参数可以知道elastic-job，如果没有配置这个参数，配置第一次初始化之后不会在更改了。
翻阅 `#JobScheduler#init` 作业初始化的源码

```java
/**
 * 初始化作业.
 */
public void init() {
    LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig);
    JobRegistry.getInstance().setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(), liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
    JobScheduleController jobScheduleController = new JobScheduleController(
            createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
    JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
    schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
    jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
}
```

首先是调用的更新配置方法，查看 `#ConfigurationService#persist `方法的实现 

```java
/**
 * 持久化分布式作业配置信息.
 * 
 * @param liteJobConfig 作业配置
 */
public void persist(final LiteJobConfiguration liteJobConfig) {
    checkConflictJob(liteJobConfig);
    if (!jobNodeStorage.isJobNodeExisted(ConfigurationNode.ROOT) || liteJobConfig.isOverwrite()) {
        jobNodeStorage.replaceJobNode(ConfigurationNode.ROOT, LiteJobConfigurationGsonFactory.toJson(liteJobConfig));
    }
}
```

这里的if条件先判断注册中心的节点是否存在，如果不存在并且 `overwrite` 的配置为true才会更新注册中心里的配置。被坑的就是这里，不管怎么修改配置甚至加上了 `overwrite` 就是不生效。所以如果第一次初始化的时候没有加 `overwrite` 参数，之后你的配置不会再被修改了，除非自己手动修改注册中心的值，或者删除注册中心的值重新初始化。所以建议这个参数最好加上，这样也可以避免修改了注册中心里的值但是和代码里的配置又对不上的问题。

---

第二个问题是追踪数据源的问题，通过程序自己创建的表名是大写的。那么问题来了，DBA那关通过不了，说大写的表名不符合规范。当时心里一万个草泥马，都快上线了，难道要卡在这里。  
和DBA撕逼之后还是妥协修改表名称为小写，谁让人家是大腿呢。  
查看数据源监听器 `#JobEventRdbListener#JobEventRdbStorage` 

```java
private static final String TABLE_JOB_EXECUTION_LOG = "job_execution_log";
    
private static final String TABLE_JOB_STATUS_TRACE_LOG = "job_status_trace_log";
    
private static final String TASK_ID_STATE_INDEX = "idx_task_state";
    
private final DataSource dataSource;
    
private DatabaseType databaseType;
```    
它已经把表名放在了常量里，修改还是比较方便的，ok，这个问题也解决了。

---

第三个问题是在监控平台查看job配置列表总是报500的错误，查看作业状态展示的实现类 `#JobStatisticsAPIImpl#getJobBriefInfo`

```java
public JobBriefInfo getJobBriefInfo(final String jobName) {
    JobNodePath jobNodePath = new JobNodePath(jobName);
    JobBriefInfo result = new JobBriefInfo();
    result.setJobName(jobName);
    String liteJobConfigJson = regCenter.get(jobNodePath.getConfigNodePath());
    if (liteJobConfigJson.isEmpty()) {
        return null;
    }
    LiteJobConfiguration liteJobConfig = LiteJobConfigurationGsonFactory.fromJson(liteJobConfigJson);
    result.setDescription(liteJobConfig.getTypeConfig().getCoreConfig().getDescription());
    result.setCron(liteJobConfig.getTypeConfig().getCoreConfig().getCron());
    result.setInstanceCount(getJobInstanceCount(jobName));
    result.setShardingTotalCount(liteJobConfig.getTypeConfig().getCoreConfig().getShardingTotalCount());
    result.setStatus(getJobStatus(jobName));
    return result;
}
```    
这里的 `liteJobConfigJson` 配置信息有可能为空，应该加个空判断

```java
if (liteJobConfigJson  == null || liteJobConfigJson.isEmpty()) 
```
但是问题是什么情况下配置的信息会为空值呢？这个问题修复后界面上只会展示elastic-job 2.0的任务节点，于是我对比了下1.0和2.0在注册中心的目录结构，发现servers的结构不一样.

![](/images/deffied.jpg)

所以如果是1.0的 job，获取子节点会是空值。所以job的控制台只能兼容到2.0。
> ps: 之前已经接入过elastic-job的系统都还停留在1.0版本

---

第四个问题是在控制台禁用 job 之后，如果服务重启任务又被开启了。

![](/images/console.png)

这里的原理和第一个问题一样，在任务初始化的时候，都会将任务为可用状态开启

```java
schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
```

### 总结
elastic-job的设计理念我觉都很巧妙，比如选举，如果是我可能会设计成同步锁，还有去中心化、分片策略、失效转移的思路也很值得去探究。  

目前的elastic-job定位是一个基于java的定时任务调度框架，仍然有很多不足的地方：

1. 异构语言不支持
	* 目前采用的无中心设计，难于支持多语言，后面需要考虑调度中心的可行性。
2. 监控体系有待提高，目前只能通过注册中心做简单的存活和数据积压监控，未来需要做的监控部分有：
	* 增加可监控维度，如作业运行时间等。
	* 基于JMX的内部状态监控。
	* 基于历史的全量数据监控，将所有监控数据通过flume等形式发到外部监控中心，提供实时分析功能。
3. 不能支持多种注册中心。
4. 需要增加任务工作流，如任务依赖，初始化任务，清理任务等。
5. 失效转移功能的实时性有待提升。
6. 缺少更多作业类型支持，如文件，MQ等类型作业的支持。
7. 缺少更多分片策略支持。




