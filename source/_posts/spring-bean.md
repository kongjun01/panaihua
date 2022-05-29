---
layout: post
title: "Spring 关于getBeansOfType获取不到实例的问题"
date: 2020-11-16
excerpt: "ElasticJob官方只有xml注册job，有很多job的时候文件非常臃肿而且不好维护，还有官方的控制台也不好用，系统多之后非常卡，页面加载的时候要一次性加载所有任务。所以我们开发出使用注解来创建job和自研的控制台。问题出在在手动触发job，通过job的class获取实例时获取不到，而且只有一个系统有问题"
index_img: /index_img/springboot.png
categories: 
- Java
tags: 
- Spring
- ElasticJob
comments: true
---

#### 问题描述

ElasticJob官方只有xml注册job，有很多job的时候文件非常臃肿而且不好维护，还有官方的控制台也不好用，系统多之后非常卡，页面加载的时候要一次性加载所有任务。所以我们开发出使用注解来创建job和自研的控制台。问题出在在手动触发job，通过job的class获取实例时获取不到，而且只有一个系统有问题。

Job的bean注册到容器的代码：

```java
DefaultListableBeanFactory defaultListableBeanFactory = (DefaultListableBeanFactory) applicationContext.getAutowireCapableBeanFactory();
defaultListableBeanFactory.registerBeanDefinition(beanId, beanDefinition);
```

获取job实例的代码：

```java
Map<String, ?> beanMap = applicationContext.getBeansOfType(jobClass);
```

上面的代码获取不到job的bean，但是这个bean确确实实的在spring容器里。

#### 进一步debug

getBeansOfType的源码：

```java
@Override
public String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
	if (!isConfigurationFrozen() || type == null || !allowEagerInit) {
		return doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, allowEagerInit);
	}
	Map<Class<?>, String[]> cache =
			(includeNonSingletons ? this.allBeanNamesByType : this.singletonBeanNamesByType);
	String[] resolvedBeanNames = cache.get(type);
	if (resolvedBeanNames != null) {
		return resolvedBeanNames;
	}
	
	// 上面内容是去缓存中获取，如果获取不到走下面的方法，同时会更新的bean的缓存
	resolvedBeanNames = doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, true);
	if (ClassUtils.isCacheSafe(type, getBeanClassLoader())) {
		cache.put(type, resolvedBeanNames);
	}
	return resolvedBeanNames;
}
```

注释的下面一行获取的对象总是空对象，所以问题出在doGetBeanNamesForType这个方法

doGetBeanNamesForType的部分源码：

```java
...

if (!isFactoryBean) {
	if (includeNonSingletons || isSingleton(beanName, mbd, dbd)) {
		// 这里是false就是没有匹配到bean
		matchFound = isTypeMatch(beanName, type, allowFactoryBeanInit);
	}
}

...

```


isTypeMatch的部分源码：

```java

...

else if (!isFactoryDereference) {
	// 这一步返回的是false，所以继续往里面debug
	if (typeToMatch.isInstance(beanInstance)) {
		// Direct match for exposed instance?
		return true;
	}
...

```

最终返回false的地方，ClassUtils源码：

```java
public static boolean isAssignable(Class<?> lhsType, Class<?> rhsType) {
		Assert.notNull(lhsType, "Left-hand side type must not be null");
		Assert.notNull(rhsType, "Right-hand side type must not be null");
		// 这一步返回的false，lhsType是LaunchClassLoader，rhsType是RestartClassLoader
		if (lhsType.isAssignableFrom(rhsType)) {
			return true;
		}
		if (lhsType.isPrimitive()) {
			Class<?> resolvedPrimitive = primitiveWrapperTypeMap.get(rhsType);
			return (lhsType == resolvedPrimitive);
		}
		else {
			Class<?> resolvedWrapper = primitiveTypeToWrapperMap.get(rhsType);
			return (resolvedWrapper != null && lhsType.isAssignableFrom(resolvedWrapper));
		}
	}
```

两个Class对象的类加载器不是同一个。全局搜索RestartClassLoader 发现其属于依赖 Spring-boot-devtools 引入的包，继承自URLClassLoader，是Spring热部署功能代码。


###### 延伸阅读

[由RestartClassLoader探索Springboot热部署](https://www.dazhuanlan.com/2019/08/30/5d6859a908b16/)
