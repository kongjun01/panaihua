---
layout: post
title: "自适应限流 netflix-concurrency-limits"
date: 2020-05-12
excerpt: "一般的限流常常需要指定一个固定值(qps)作为限流开关的阈值，这个值一是靠经验判断，二是靠通过大量的测试数据得出。但这个阈值，在流量激增、系统自动伸缩或者某某commit了一段有毒代码后就有可能变得不那么合适了。并且一般业务方也不太能够正确评估自己的容量，去设置一个合适的限流阈值。"
categories: 
- Java
tags: 
- 限流
comments: true
---
作为应对高并发的手段之一，限流并不是一个新鲜的话题了。从Guava的Ratelimiter到Hystrix，以及Sentinel都可作为限流的工具。

### 自适应限流

一般的限流常常需要指定一个固定值(qps)作为限流开关的阈值，这个值一是靠经验判断，二是靠通过大量的测试数据得出。但这个阈值，在流量激增、系统自动伸缩或者某某commit了一段有毒代码后就有可能变得不那么合适了。并且一般业务方也不太能够正确评估自己的容量，去设置一个合适的限流阈值。

而此时自适应限流就是解决这样的问题的，限流阈值不需要手动指定，也不需要去预估系统的容量，并且阈值能够随着系统相关指标变化而变化。

自适应限流算法借鉴了TCP拥塞算法，根据各种指标预估限流的阈值，并且不断调整。大致获得的效果如下:

![](/images/20191104083803.jpg)

从图上可以看到，首先以一个降低的初始并发值发送请求，同时通过增大限流窗口来探测系统更高的并发性。而一旦延迟增加到一定程度了，又会退回到较小的限流窗口。循环往复持续探测并发极限，从而产生类似锯齿状的时间关系函数。

### TCP Vegas

vegas是一种主动调整cwnd的拥塞控制算法，主要是设置两个阈值alpha 和 beta，然后通过计算目标速率和实际速率的差diff，再比较差diff与alpha和beta的关系，对cwnd进行调节。伪代码如下：
```
diff = cwnd*(1-baseRTT/RTT)
if (diff < alpha)
set: cwnd = cwnd + 1 
else if (diff >= beta)
set: cwnd = cwnd - 1
else
set: cwnd = cwnd
```

其中baseRTT指的是测量的最小往返时间，RTT指的是当前测量的往返时间，cwnd指的是当前的TCP窗口大小。通常在tcp中alpha会被设置成2-3，beta会被设置成4-6。这样子，cwnd就保持在了一个平衡的状态。

##### netflix-concuurency-limits

concuurency-limits是netflix推出的自适应限流组件，借鉴了TCP相关拥塞控制算法，主要是根据请求延时，及其直接影响到的排队长度来进行限流窗口的动态调整。

##### alpha , beta & threshold

vegas算法实现在了VegasLimit类中。先看一下初始化相关代码:


```java
private int initialLimit = 20;
    private int maxConcurrency = 1000;
    private MetricRegistry registry = EmptyMetricRegistry.INSTANCE;
    private double smoothing = 1.0;
    
    private Function<Integer, Integer> alphaFunc = (limit) -> 3 * LOG10.apply(limit.intValue());
    private Function<Integer, Integer> betaFunc = (limit) -> 6 * LOG10.apply(limit.intValue());
    private Function<Integer, Integer> thresholdFunc = (limit) -> LOG10.apply(limit.intValue());
    private Function<Double, Double> increaseFunc = (limit) -> limit + LOG10.apply(limit.intValue());
    private Function<Double, Double> decreaseFunc = (limit) -> limit - LOG10.apply(limit.intValue());
```

这里首先定义了一个初始化值initialLimit为20，以及极大值maxConcurrency1000。其次是三个
阈值函数alphaFunc，betaFunc以及thresholdFunc。最后是两个增减函数increaseFunc和decreaseFunc。
函数都是基于当前的并发值limit做运算的。

- alphaFunc可类比vegas算法中的alpha，此处的实现是3*log limit。limit值从初始20增加到极大1000时候，相应的alpha从3.9增加到了9。
- betaFunc则可类比为vegas算法中的beta，此处的实现是6*log limit。limit值从初始20增加到极大1000时候，相应的alpha从7.8增加到了18。
- thresholdFunc算是新增的一个函数，表示一个较为初始的阈值，小于这个值的时候limit会采取激进一些的增量算法。这里的实现是1倍的log limit。mit值从初始20增加到极大1000时候，相应的alpha从1.3增加到了3。

这三个函数值可以认为确定了动态调整函数的四个区间范围。当变量queueSize = limit × (1 − RTTnoLoad/RTTactual)落到这四个区间的时候应用不同的调整函数。

##### 变量queueSize

其中变量为queueSize，计算方法即为limit × (1 − RTTnoLoad/RTTactual)，为什么这么计算其实稍加领悟一下即可。

![](/images/2019110483804.jpg)

我们把系统处理请求的过程想象为一个水管，到来的请求是往这个水管灌水，当系统处理顺畅的时候，请求不需要排队，直接从水管中穿过，这个请求的RT是最短的，即RTTnoLoad；反之，当请求堆积的时候，那么处理请求的时间则会变为：排队时间+最短处理时间，即RTTactual = inQueueTime + RTTnoLoad。而显然排队的队列长度为
总排队时间/每个请求的处理时间及queueSize = (limit * inQueueTime) / (inQueueTime + RTTnoLoad) = limit × (1 − RTTnoLoad/RTTactual)。
再举个栗子，因为假设当前延时即为最佳延时，那么自然是不用排队的，即queueSize=0。而假设当前延时为最佳延时的一倍的时候，可以认为处理能力折半，100个流量进来会有一半即50个请求在排队，及queueSize= 100 * (1 − 1/2)=50。

##### 动态调整函数

调整函数中最重要的即增函数与减函数。从初始化的代码中得知，增函数increaseFunc实现为limit+log limit，减函数decreaseFunc实现为limit-log limit，相对来说增减都是比较保守的。

看一下应用动态调整函数的相关代码：

```java
private int updateEstimatedLimit(long rtt, int inflight, boolean didDrop) {
    final int queueSize = (int) Math.ceil(estimatedLimit * (1 - (double)rtt_noload / rtt));

    double newLimit;
    // Treat any drop (i.e timeout) as needing to reduce the limit
    // 发现错误直接应用减函数decreaseFunc
    if (didDrop) {
        newLimit = decreaseFunc.apply(estimatedLimit);
    // Prevent upward drift if not close to the limit
    } else if (inflight * 2 < estimatedLimit) {
        return (int)estimatedLimit;
    } else {
        int alpha = alphaFunc.apply((int)estimatedLimit);
        int beta = betaFunc.apply((int)estimatedLimit);
        int threshold = this.thresholdFunc.apply((int)estimatedLimit);

        // Aggressive increase when no queuing
        if (queueSize <= threshold) {
            newLimit = estimatedLimit + beta;
        // Increase the limit if queue is still manageable
        } else if (queueSize < alpha) {
            newLimit = increaseFunc.apply(estimatedLimit);
        // Detecting latency so decrease
        } else if (queueSize > beta) {
            newLimit = decreaseFunc.apply(estimatedLimit);
        // We're within he sweet spot so nothing to do
        } else {
            return (int)estimatedLimit;
        }
    }

    newLimit = Math.max(1, Math.min(maxLimit, newLimit));
    newLimit = (1 - smoothing) * estimatedLimit + smoothing * newLimit;
    if ((int)newLimit != (int)estimatedLimit && LOG.isDebugEnabled()) {
        LOG.debug("New limit={} minRtt={} ms winRtt={} ms queueSize={}",
                (int)newLimit,
                TimeUnit.NANOSECONDS.toMicros(rtt_noload) / 1000.0,
                TimeUnit.NANOSECONDS.toMicros(rtt) / 1000.0,
                queueSize);
    }
    estimatedLimit = newLimit;
    return (int)estimatedLimit;
}
```

动态调整函数规则如下：

- 当变量queueSize < threshold时，选取较激进的增量函数，newLimit = limit+beta
- 当变量queueSize < alpha时，需要增大限流窗口，选择增函数increaseFunc，即newLimit = limit + log limit
- 当变量queueSize处于alpha，beta之间时候，limit不变
- 当变量queueSize大于beta时候，需要收拢限流窗口，选择减函数decreaseFunc，即newLimit = limit - log limit

##### 平滑递减 smoothingDecrease

注意到可以设置变量smoothing，这里初始值为1，表示平滑递减不起作用。如果有需要的话可以按需设置，比如设置smoothing为0.5时候，那么效果就是采用减函数decreaseFunc时候效果减半，实现方式为newLimitAfterSmoothing = 0.5 newLimit + 0.5 limit。






