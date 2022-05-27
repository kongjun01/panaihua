---
layout: post
title: "mybaits 关于游标状态closed的问题"
date: 2017-06-06
excerpt: "由于数据不断增加，查询的时候如果一次性加载出来，会导致mybaits的mapper对象非常大，可以选择分页但是觉得太麻烦了所以选择了游标。之前使用游标是jdbc自带的，没有遇到读取数据游标状态会closed的情况。"
categories: 
- Java
tags: 
- Mybaits
comments: true
---

由于数据不断增加，查询的时候如果一次性加载出来，会导致mybaits的mapper对象非常大，可以选择分页但是觉得太麻烦了所以选择了游标。之前使用游标是jdbc自带的，没有遇到读取数据游标状态会closed的情况。后来不断debug发现是mybaits封装SqlSessionTemplate的问题。现把解决的过程记录下来。

#### 下面是SqlSessionTemplate执行语句的主要内容

```java
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```

这个方法的最后会关闭session，问题就出在这里，当游标读完的时候也会被关闭。但不是我们想要的结果。再看看SqlSessionUtils.closeSqlSession里的方法。

```java
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    if ((holder != null) && (holder.getSqlSession() == session)) {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Releasing transactional SqlSession [" + session + "]");
      }
      holder.released();
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Closing non transactional SqlSession [" + session + "]");
      }
      session.close();
    }
  }
```
在第5行的判断，大概意思是如果是同一事务里的session会继续执行下去。
所以解决这个问题就是在读取游标的外面加一层事务控制。感觉没有jdbc读取游标的方法好用，还多此一举了，先这么办吧。之后如果有时间改造一个读取游标的方法。


