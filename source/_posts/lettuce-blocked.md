---
layout: post
title: "springboot2 整合lettuce启动卡住的问题"
date: 2020-11-26
excerpt: "EasyCache升级兼容 `Springboot2`，有个业务系统启动总是会卡住，最后抛出超时异常，springboot 版本是 2.2.x，springCloudVersion 版本是 2.2.x, lettuce版本是5.2.x，如果使用jedis客户端没有，所以问题一定是出在lettuce。"
categories: 
- Java
tags:
- Springboot
- Lettuce
comments: true
---

### 前言

EasyCache升级兼容 `Springboot2`，有个业务系统启动总是会卡住，最后抛出超时异常，如下：

```java
java.util.concurrent.TimeoutException: null
	at java.util.concurrent.FutureTask.get(FutureTask.java:205)
.....
```

springboot 版本是 2.2.x，springCloudVersion 版本是 2.2.x, lettuce版本是5.2.x，如果使用jedis客户端没有，所以问题一定是出在lettuce。

### 分析原因

如果是线上发生这个问题会使用 jstack 查看线程的情况，在本地idea调试就更加方便了，查看线程发现lettuce的线程被Blocked，dump出的部分信息如下：

```
"lettuce-kqueueEventLoop-7-1@14257" daemon prio=5 tid=0x4c nid=NA waiting for monitor entry
  java.lang.Thread.State: BLOCKED
 waiting for main@1 to release lock on <0x38a5> (a java.util.concurrent.ConcurrentHashMap)
  at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:208)
  at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321)
	  ....
```


看第一行的报错是在获取Bean的时候阻塞了，说明有地方获取Bean的时候没有释放锁。在这地方打断点发现是 spring-cloud-sleuth 的 `SamplerAutoConfiguration`获取bean的时候有锁没有释放。源代码如下

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(type = "org.springframework.cloud.context.scope.refresh.RefreshScope")
protected static class RefreshScopedSamplerConfiguration {
	
	@Bean
	@RefreshScope
	@ConditionalOnMissingBean
	public Sampler defaultTraceSampler(SamplerProperties config) {
		return samplerFromProps(config);
	}

}
```

@RefreshScope 获取代理类的时候如果是@PostConstruct的方法，bean是加载不到，所以导致一直没有释放锁。所以猜想，容器还没有启动完成的时候，有地方调用了lettuce的Bean,导致循环依赖。

### 坑的复现及解决办法

运行下面这段代码，错误就出现了，和业务系统出现的问题一模一样，也验证了上面的猜想。解决办法是在容器启动之后在调用init方法。（实测使用InitializingBean时也会出现该问题）

```java
@Service
public class SpringDataTestService {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    //@EventListener(MainContextRefreshedEvent.class)
    @PostConstruct
    public void init() {
        String s = stringRedisTemplate.opsForValue().get("gateway:ab-test:config");
        System.out.println(s);
    }
}
```

