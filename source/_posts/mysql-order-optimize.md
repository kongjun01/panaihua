---
layout: post
title: "MySQL 索引提高优化Order By"
date: 2016-04-08
excerpt: "在数据库中我们一般都会对一些字段进行索引操作，这样可以提升数据的查询速度，同时提高数据库的性能像order by ,group by前都需要索引哦。"
index_img: /index_img/mysql.png
categories: 
- Mysql
tags: 
- Mysql
comments: true
---

在数据库中我们一般都会对一些字段进行索引操作，这样可以提升数据的查询速度，同时提高数据库的性能像order by ,group by前都需要索引哦。  
>首先我们要注意一下  
>1. mysql一次查询只能使用一个索引。如果要对多个字段使用索引，建立复合索引。  
2. 在ORDER BY操作中，MySQL只有在排序条件不是一个查询条件表达式的情况下才使用索引。

## 关于索引一些说法
MySQL索引通常是被用于提高WHERE条件的数据行匹配或者执行联结操作时匹配其它表的数据行的搜索速度。  
MySQL也能利用索引来快速地执行ORDER BY和GROUP BY语句的排序和分组操作。  
### 通过索引优化来实现MySQL的ORDER BY语句优化

* ORDER BY的索引优化  
如果一个SQL语句形如：SELECT [column1],[column2],…. FROM [TABLE] ORDER BY [sort];  
在[sort]这个栏位上建立索引就可以实现利用索引进行order by 优化。  

* WHERE + ORDER BY的索引优化  
例如：SELECT [column1],[column2],…. FROM [TABLE] WHERE [columnX] = [value] ORDER BY [sort];建立一个联合索引(columnX,sort)来实现order by 优化。  
注意：如果columnX对应多个值，如下面语句就无法利用索引来实现order by的优化
SELECT [column1],[column2],…. FROM [TABLE] WHERE [columnX] IN ([value1],[value2],…) ORDER BY[sort];  

* WHERE+ 多个字段ORDER BY  
SELECT * FROM [table] WHERE uid=1 ORDER x,y LIMIT 0,10;  
建立索引(uid,x,y)实现order by的优化,比建立(x,y,uid)索引效果要好得多。  
### MySQL Order By不能使用索引来优化排序的情况

* 对不同的索引键做 ORDER BY ：(key1,key2分别建立索引)  
  SELECT * FROM t1 ORDER BY key1, key2;

* 在非连续的索引键部分上做 ORDER BY：(key_part1,key_part2建立联合索引;key2建立索引)  
SELECT * FROM t1 WHERE key2=constant ORDER BY key_part2;

* 同时使用了 ASC 和 DESC：(key_part1,key_part2建立联合索引)  
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC;

* 用于搜索记录的索引键和做 ORDER BY 的不是同一个：(key1,key2分别建立索引)  
SELECT * FROM t1 WHERE key2=constant ORDER BY key1;

* 如果在WHERE和ORDER BY的栏位上应用表达式(函数)时，则无法利用索引来实现order by的优化  
SELECT * FROM t1 ORDER BY YEAR(logindate) LIMIT 0,10;

### MySQL支持很多数据类型，选择合适的数据类型存储数据对性能有很大的影响。
通常来说，可以遵循以下一些指导原则：

* 越小的数据类型通常更好：越小的数据类型通常在磁盘、内存和CPU缓存中都需要更少的空间，处理起来更快。
* 简单的数据类型更好：整型数据比起字符，处理开销更小，因为字符串的比较更复杂。在MySQL中，应该用内置的日期和时间数据类型，而不是用字符串来存储时间；以及用整型数据类型存储IP地址。
* 尽量避免NULL：应该指定列为NOT NULL，除非你想存储NULL。在MySQL中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。你应该用0、一个特殊的值或者一个空串代替空值。
