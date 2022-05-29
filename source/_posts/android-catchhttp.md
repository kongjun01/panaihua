---
layout: post
title: "Android 破解APP抓包限制(绕开https的SSL Pinning)"
date: 2020-12-02
excerpt: "这篇文章主要想解决的问题是，在对安卓手机APP抓包时，出现的HTTPS报文通过MITM代理后证书不被信任的问题。如果有些https，在之前设置了好各种证书和配置后，看到的unknown..而无法看到我们希望的明文数据"
index_img: /index_img/android.jpg
categories: 
- Android
tags: 
- 爬虫
comments: true
---

### 介绍

> 这里写下来是为了防止自己遗忘的，毕竟去刷机也不是经常有的事

这篇文章主要想解决的问题是，在对安卓手机APP抓包时，出现的HTTPS报文通过MITM代理后证书不被信任的问题。如果有些https，在之前设置了好各种证书和配置后，看到的：

- 要么是unknown
- 要么是：加密的乱码
- 要么是：报错无法抓包

而无法看到我们希望的明文数据，则：

最大可能是，对方用了https的 `SSL pinning`

#### 什么是SSL Pinning

>SSL pinning = 证书绑定 = SSL证书绑定

对方的app内部，只允许，承认其自己的，特定的证书

导致此处MITM的证书不识别，不允许

导致MITM无法解密看到https的明文数据


#### Android 7.0之后系统如何破解https的ssl pinning

对于Android 7.0 (API 24) 之后，做了些改动，使得系统安全性增加了，导致：

- APP 默认不信任用户域的证书。之前把MITM的ssl证书，安装到 受信任的凭据 -> 用户 就没用了，因为不受信任了。只信任（安装到）系统域的证书

导致无法抓包https，抓出来的https的请求，都是加了密的，无法看到原文了。

对此，总结出相关解决思路和方案：

- （努力想办法）让系统信任Charles的ssl证书
	- 作为app的开发者自己：改自己的app的配置，允许https抓包。前提是得到或本身有app的源码
	- 把证书放到受系统信任的系统证书中去。前提是手机已root

- 绕开https不去校验
	- 借助于其他（JustTrustMe等）工具绕开https的校验。需要借助其他XPosed等框架配合才可以 （这里只介绍这一种方案）


### 刷机（ROOT）

网上有很多刷机教程，我试过小米、oppo、华为、一加，除了一加其它root都需要很多门槛，oppo需要深度测试申请，审核估计都要15天，华为需要解锁码，小米也要申请，而一加什么门槛都没有，而且官方论坛一堆小白刷机工具，一键就能搞定。所以极力推荐一加手机。

这里推荐 @千古风流帝 的 [工具箱](https://www.oneplusbbs.com/thread-4701007-1.html)，该工具箱里包含了解锁、TWRP、Root 等一系列无脑操作。按照软件的教程，先解锁在ROOT，它会自动刷入 [Magisk](https://github.com/topjohnwu/Magisk)，刷完之后多一个MagiskManager的APP。至此ROOT已经成完成

#### 安装太极·Magisk

按照官方教程刷入，这里不在赘述，[教程地址](https://github.com/taichi-framework/TaiChi/wiki/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8)

#### 安装JustTrustMe模块
 
 [JustTrustMe下载地址](https://github.com/taichi-framework/TaiChi/issues/538)
 
 ok，这样就可以正常抓包了，测试手机是一加5T，测试APP是抖音。



