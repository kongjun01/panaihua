---
layout: post
title: "Gradle SNAPSHOT版本不能更新的问题"
date: 2017-03-29
excerpt: "团队里使用SNAPSHOT版本每次修改都要重新发布个版本，本身就是快照版，为什么都要升级呢，还不如用release版本呢。之前一直使用maven没有发现这个问题，是因为gradle为了加快构建的速度,对jar包默认会缓存24小时，缓存之后就不在请求远程仓库了。"
categories: 
- Java
tags: 
- Gradle
comments: true
---

团队里使用SNAPSHOT版本每次修改都要重新发布个版本，本身就是快照版，为什么都要升级呢，还不如用release版本呢。之前一直使用maven没有发现这个问题，是因为gradle为了加快构建的速度,对jar包默认会缓存24小时，缓存之后就不在请求远程仓库了。  

#### 在Gradle里设置本地缓存的更新策略

```java
configurations.all {  
	// check for updates every build   
	resolutionStrategy.cacheChangingModulesFor  0,'seconds'  
}
```

上面的方法别人可能会有效果，但是我的就是怎么都没效果。google了半天，发现这个插件效果没有应用到子项目里，参考地址：https://github.com/spring-gradle-plugins/dependency-management-plugin/issues/38

```java
dependencyManagement {
    resolutionStrategy {
        cacheChangingModulesFor 0, 'seconds'
    }
}
```
好了，看下gradle的构建日志，去远程仓库获取了。