---
layout: post
title: "APISIX 与现有网关以及 SpringCloudGateWay 压测报告对比"
date: 2021-07-22
excerpt: "针对接口分别执行线程总数 400（jemter1台）、800（jemter2台） 进行压力测试，并对产生的每秒TPS，响应时间（min，ave，max）及错误率进行统计"
categories: 
- 网关
tags: 
- APISIX
comments: true
---

#### 压测环境

- 测试主机：阿里云 8 vCPU 16 GiB 分别1台网关，1台后端

- 压测工具：jemter

- 压测说明：

针对接口分别执行线程总数 400（jemter1台）、800（jemter2台） 进行压力测试，并对产生的每秒TPS，响应时间（min，ave，max）及错误率进行统计。后端有一个/hello接口，请求方式GET。网关反向代理到后端.

APISIX模型：

![](/images/apisix-1.png)

zuul-server公司现有网关模型:

![](/images/apisix-2.png)

全新的springCloudGateway(没有开发任何功能):

![](/images/apisix-3.png)


#### 8000QPS的指标对比

##### APISIX未开启证书

![](/images/apisix-4.png)


##### ZUUL-SERVER

![](/images/apisix-5.png)

##### SpringCloudGateway

![](/images/apisix-6.png)

##### APISIX开启证书

平均RT压了多次都是0，可能被四舍五入了

![](/images/apisix-7.png)

#### 16000QPS的指标对比

##### APISIX未开启https

![](/images/apisix-8.png)


##### APISIX开启HTTPS

![](/images/apisix-9.png)

##### SpringCloudGateway

![](/images/apisix-10.png)

#### 30000QPS的指标对比

##### APISIX未开启HTTPS

![](/images/apisix-11.png)

##### APISIX开启HTTPS

![](/images/apisix-12.png)


#### 压测结果

APISIX: 开启插件和不开启插件的性能差不了多少，开启HTTPS的功能之后，16000QPS的RT无差，3万QPS之后RT会增加一倍。

APISIX的3万QPS的情况下，cpu使用率在30%～40%；

zuul-server的8000QPS已经是极限；SpringCloudGateWay只能支撑到16000QPS。





