---
layout: post
title: "MySQL Explain总结"
date: 2016-05-10
excerpt: "MySQL 查询优化器有几个目标,但是其中最主要的目标是尽可能地使用索引,并且使用最严格的索引来消除尽可能多的数据行。最终目标是提交 SELECT 语句查找数据行,而不是排除数据行。优化器试图排除数据行的原因在于它排除数据行的速度越快,那么找到与条件匹配的数据行也就越快。如果能够首先进行最严格的测试,查询就可以执行地更快."
categories: 
- Mysql
tags: 
- Mysql
comments: true
---

## MySQL 查询优化器是如何工作的
MySQL 查询优化器有几个目标,但是其中最主要的目标是尽可能地使用索引,并且使用最严格的索引来消除尽可能多的数据行。最终目标是提交 SELECT 语句查找数据行,而不是排除数据行。优化器试图排除数据行的原因在于它排除数据行的速度越快,那么找到与条件匹配的数据行也就越快。如果能够首先进行最严格的测试,查询就可以执行地更快。

## Explain 的每个输出列的含义
#### id
MySQL Query Optimizer 选定的执行计划中查询的序列号。表示查询中执行 select 子句或操作表的顺序,id 值越大优先级越高,越先被执行。id 相同,执行顺序由上至下。
#### select_type 
查询类型  
*SIMPLE* : 简单的 select 查询,不使用 union 及子查询  
*PRIMARY* : 最外层的 select 查询  
*UNION* : UNION 中的第二个或随后的 select 查询,不 依赖于外部查询的结果集  
*DEPENDENT UNION* : UNION 中的第二个或随后的 select 查询,依 赖于外部查询的结果集  
*SUBQUERY* : 子查询中的第一个 select 查询,不依赖于外 部查询的结果集  
*DEPENDENT SUBQUERY* : 子查询中的第一个 select 查询,依赖于外部 查询的结果集  
*DERIVED* : 用于 from 子句里有子查询的情况。 MySQL 会 递归执行这些子查询, 把结果放在临时表里。  
*UNCACHEABLE SUBQUERY* : 结果集不能被缓存的子查询,必须重新为外 层查询的每一行进行评估。  
*UNCACHEABLE UNION* : UNION 中的第二个或随后的 select 查询,属 于不可缓存的子查询  
#### table
顾名思义，就是查询的表
#### type
输出行所引用的表（重要的项,显示连接使用的类型,按最优到最差的类型排序）  
*system* ： 表仅有一行(=系统表)。这是 const 连接类型的一个特例。  
*const* ： 用于用常数值比较 PRIMARY KEY 时。当 查询的表仅有一行时,使用 System。  
*eq\_ref* ： 用于用常数值比较 PRIMARY KEY 时。当 查询的表仅有一行时,使用 System。  
*ref* ： 连接不能基于关键字选择单个行,可能查找到多个符合条件的行。叫做 ref 是因为索引要 跟某个参考值相比较。这个参考值或者是一 个常数,或者是来自一个表里的多表查询的 结果值。  
*ref\_or\_null* ： 如同 ref, 但是 MySQL 必须在初次查找的结果 里找出 null 条目,然后进行二次查找。  
*index\_merge* ： 说明索引合并优化被使用了。  
*unique\_subquery* ： 在某些 IN 查询中使用此种类型,而不是常规的。  
*index\_subquery* ： 在某些IN查询中使用此种类型,与unique\_subquery类似,但是查询的是非唯一 性索   
*range* ： 只检索给定范围的行,使用一个索引来选择 行。key 列显示使用了哪个索引。当使用=、 <>、>、>=、<、<=、IS NULL、<=>、BETWEEN 或者 IN 操作符,用常量比较关键字列时,可以使用 range。  
*index* ： 全表扫描,只是扫描表的时候按照索引次序 进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大。  
*all* ： 最坏的情况,从头到尾全表扫描。
MySQL一次查询只能使用一个索引（多个表查询时就是表的数量），所以MySQL在选择索引的时候会按照上面的顺序使用。
比如：（每个字段都有索引）  

```
select * from tbl_order where  id >= 1000 and name = "zhangsan" limit 0,10 
```
>看似会使用主键的索引，其实不然。”=“是”ref“的类型， ”>=“是”range“类型，优先级最低，所以MySQL会使用name的索引。

#### possible\_keys
指出 MySQL 能在该表中使用哪些索引有助于 查询。如果为空,说明没有可用的索引。
#### key
MySQL 实际从 possible_key 选择使用的索引。 如果为 NULL,则没有使用索引。很少的情况 下,MYSQL 会选择优化不足的索引。这种情况下,可以在 SELECT 语句中使用 USE INDEX (indexname)来强制使用一个索引或者用 IGNORE INDEX(indexname)来强制 MYSQL 忽略索引
#### key\_len
使用的索引的长度。在不损失精确性的情况 下,长度越短越好。
#### ref
显示索引的哪一列被使用了
#### rows
MySQL 认为必须检查的用来返回请求数据的行数
#### extra
出现以下两项意味着 MYSQL 根本不能使用索引,效率会受到重大影响。应尽可能对此进行优化。  
*Using filesort* : 表示 MySQL 会对结果使用一个外部索引排序,而不是从表里按索引次序读到相关内容。可能在内存或者磁盘上进行排序。MySQL 中无法利用索引完成的排序操作称为“文件排序”  
*Using temporary* : 表示 MySQL 在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 group by。

MySql 中的 explain 语法可以帮助我们改写查询,优化表的结构和索引的设置,从而最大地提高查询效率。当然,在大规模数据量时,索引的建立和维护的代价也是很高的,往往需要较长的时间和较大的空间,如果在不同的列组合上建立索引,空间的开销会更大。因此索引最好设置在需要经常查询的字段中。














