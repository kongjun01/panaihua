---
layout: post
title: "J.U.C之AQS原理-CLH队列"
date: 2020-01-09
excerpt: "CLH 同步队列是一个 FIFO 双向队列，AQS 依赖它来完成同步状态的管理：当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程。当同步状态释放时，会将某个节点唤醒（是否首节点取决于公平锁/非公平锁），使其再次尝试获取同步状态。"
categories: 
- Java
tags: 
- AQS
comments: true
---

### 简介

CLH 同步队列是一个 FIFO 双向队列，AQS 依赖它来完成同步状态的管理：

- 当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程。
- 当同步状态释放时，会将某个节点唤醒（是否首节点取决于公平锁/非公平锁），使其再次尝试获取同步状态。


### Node

> CLH队列由Node对象组成，Node是AQS中的内部类。

![](/images/5cbd6a30badb8.jpeg)

上图直观的向我们展示了节点的组织状态，我们可以看看node节点的源代码。

node节点作为CLH队列的一个节点，有着5条属性，分别是waitStatus、prev、next、thread、nextWater。

##### waitStatus

> waitStatus是当前节点的一个等待状态标志位，该标志位决定了该节点在当前情况下处于何种状态。

变量waitStatus则表示当前被封装成Node结点的等待状态，共有4种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE。不用再说了，直接看注释吧。

```java
/** waitStatus value to indicate thread has cancelled */  
static final int CANCELLED = 1;//表示等待锁的线程，被取消
/** waitStatus value to indicate successor's thread needs unparking */
static final int SIGNAL = -1;//表示后继线程需要被唤醒
/** waitStatus value to indicate thread is waiting on condition */  
static final int CONDITION = -2;//表示在等待条件  
/** 
 * waitStatus value to indicate the next acquireShared should 
 * unconditionally propagate 
 */  
static final int PROPAGATE = -3;//表示下一个获取共享锁的线程，无条件传递获取 
// 小于0时是有效状态，说明线程处于可以被唤醒的状态，大于0时取消状态，说明该线程中断或者等待超时，需要移除该线程。
```

- CANCELLED(1)：表示当前节点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的节点将不会再变化
- SIGNAL(-1)：表示后继节点在等待当前节点唤醒。后继节点入队后进入休眠状态之前，会将前驱节点的状态更新为SIGNAL
- CONDITION(-2)：表示节点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的节点将从等待队列转移到同步队列中，等待获取同步锁
- PROPAGATE(-3)：共享模式下，前驱节点不仅会唤醒其后继节点，同时也可能会唤醒后继的后继节点

##### nextWaiter

AQS中阻塞队列采用的是用双向链表保存，用prve和next相互链接。而AQS中条件队列是使用单向列表保存的，用nextWaiter来连接。阻塞队列和条件队列并不是使用的相同的数据结构。
在Node节点的源码中有两个常量属性

```java
// 共享模式
static final Node SHARED = new Node();
// 独占模式
static final Node EXCLUSIVE = null;
// 其他模式
// 其他非空值：条件等待节点（调用Condition的await方法的时候
```

nextWaiter实际上标记的就是在该节点唤醒后依据该节点的状态判断是否依据条件唤醒下一个节点。

|  nextWaiter状态标志   | 说明 |
|  ----  | ----  |
|  SHARED(共享模式)  | 直接唤醒下一个节点  |
|  EXCLUSIVE(独占模式)  | 等待当前线程执行完成后再唤醒  |


### 入队

上面了解了同步队列的结构， 在分析其入列操作在简单不过。无非就是将tail（使用CAS保证原子操作）指向新节点，新节点的prev指向队列中最后一节点（旧的tail节点），原队列中最后一节点的next节点指向新节点以此来建立联系。

![](/images/5cbd6a7a93674.jpeg)

具体实现查看`‌addWaiter()`源码。

### 出队

同步队列（CLH）遵循FIFO，首节点是获取同步状态的节点，首节点的线程释放同步状态后，将会唤醒它的后继节点（next），而后继节点将会在获取同步状态成功时将自己设置为首节点，这个过程非常简单。如下图

![](/images/5cbd6b431be55.jpeg)

设置首节点是通过获取同步状态成功的线程来完成的（获取同步状态是通过CAS来完成），只能有一个线程能够获取到同步状态，因此设置头节点的操作并不需要CAS来保证，只需要将首节点设置为其原首节点的后继节点并断开原首节点的next（等待GC回收）应用即可。


### 总结
CLH阻塞队列采用的是双向链表队列，头部节点默认获取资源获得执行权限。后续节点不断自旋方式查询前置节点是否执行完成，直到头部节点执行完成将自己的waitStatus状态修改以通知后续节点可以获取资源执行。CLH锁是一个有序的无饥饿的公平锁。