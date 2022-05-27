---
layout: post
title: "JVM GC 策略&内存申请、对象衰老"
date: 2016-04-21
excerpt: "JVM里的GC(Garbage Collection)的算法有很多种，如标记清除收集器，压缩收集器，分代收集器等等,详见HotSpot VM GC 的种类"
categories: 
- Java
tags: 
- GC
comments: true
---

JVM里的GC(Garbage Collection)的算法有很多种，如标记清除收集器，压缩收集器，分代收集器等等,详见HotSpot VM GC 的种类。  
>现在比较常用的是分代收集（generational collection,也是SUN VM使用的,J2SE1.2之后引入），即将内存分为几个区域，将不同生命周期的对象放在不同区域里:young generation，tenured generation和permanet generation。绝大部分的objec被分配在young generation(生命周期短)，并且大部分的object在这里die。当young generation满了之后，将引发minor collection(YGC)。在minor collection后存活的object会被移动到tenured generation(生命周期比较长)。最后，tenured generation满之后触发major collection。major collection（Full gc）会触发整个heap的回收，包括回收young generation。permanet generation区域比较稳定，主要存放classloader信息。  
young generation有eden、2个survivor 区域组成。其中一个survivor区域一直是空的，是eden区域和另一个survivor区域在下一次copy collection后活着的objecy的目的地。object在survivo区域被复制直到转移到tenured区。  
我们要尽量减少 Full gc 的次数(tenured generation 一般比较大,收集的时间较长,频繁的Full gc会导致应用的性能收到严重的影响)。  

## 堆内存GC
JVM(采用分代回收的策略)，用较高的频率对年轻的对象(young generation)进行YGC，而对老对象(tenured generation)较少(tenured generation 满了后才进行)进行Full GC。这样就不需要每次GC都将内存中所有对象都检查一遍。
## 非堆内存不GC
GC不会在主程序运行期对PermGen Space进行清理，所以如果你的应用中有很多CLASS(特别是动态生成类，当然permgen space存放的内容不仅限于类)的话,就很可能出现PermGen Space错误。
### 内存申请过程

1. JVM会试图为相关Java对象在Eden中初始化一块内存区域；
2. 当Eden空间足够时，内存申请结束。否则到下一步；
3. JVM试图释放在Eden中所有不活跃的对象（minor collection），释放后若Eden空间仍然不足以放入新对象，则试图将部分Eden中活跃对象放入Survivor区；
4. Survivor区被用来作为Eden及old的中间交换区域，当OLD区空间足够时，Survivor区的对象会被移到Old区，否则会被保留在Survivor区；
5. 当old区空间不够时，JVM会在old区进行major collection；
6. 完全垃圾收集后，若Survivor及old区仍然无法存放从Eden复制过来的部分对象，导致JVM无法在Eden区为新对象创建内存区域，则出现”Out of memory错误”；

### 对象衰老过程
1. 新创建的对象的内存都分配自eden。Minor collection的过程就是将eden和在用survivor space中的活对象copy到空闲survivor space中。对象在young generation里经历了一定次数(可以通过参数配置)的minor collection后，就会被移到old generation中，称为tenuring。

2. GC触发条件
<table border="1" cellspacing="0">
<tbody>
<tr>
<td valign="top"><strong>GC类型</strong></td>
<td valign="top"><strong>触发条件</strong></td>
<td valign="top"><strong>触发时发生了什么</strong></td>
<td valign="top"><strong>注意</strong></td>
<td valign="top"><strong>查看方式</strong></td>
</tr>
<tr>
<td valign="top">YGC</td>
<td valign="top">eden空间不足</td>
<td valign="top">清空Eden+from survivor中所有no ref的对象占用的内存<br>
将eden+from sur中所有存活的对象copy到to sur中<br>
一些对象将晋升到old中:<br>
to sur放不下的<br>
存活次数超过turning threshold中的<br>
重新计算tenuring threshold(serial parallel GC会触发此项)<p></p>
<p>重新调整Eden 和from的大小(parallel GC会触发此项)</p></td>
<td valign="top">全过程暂停应用<br>
是否为多线程处理由具体的GC决定</td>
<td valign="top">jstat –gcutil<br>
gc log</td>
</tr>
<tr>
<td valign="top">FGC</td>
<td valign="top">old空间不足<br>
perm空间不足<br>
显示调用System.GC, RMI等的定时触发<br>
YGC时的悲观策略<br>
dump live的内存信息时(jmap –dump:live)</td>
<td valign="top">清空heap中no ref的对象<br>
permgen中已经被卸载的classloader中加载的class信息<p></p>
<p>如配置了CollectGenOFirst,则先触发YGC(针对serial GC)<br>
如配置了ScavengeBeforeFullGC,则先触发YGC(针对serial GC)</p></td>
<td valign="top" width="155">全过程暂停应用<br>
是否为多线程处理由具体的GC决定<p></p>
<p>是否压缩需要看配置的具体GC</p></td>
<td valign="top">jstat –gcutil<br>
gc log</td>
</tr>
</tbody>
</table>
permanent generation空间不足会引发Full GC,仍然不够会引发PermGen Space错误。








