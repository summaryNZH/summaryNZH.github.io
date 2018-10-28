---
layout:     post
title:      分页优化技巧-延迟关联-索引覆盖
subtitle:   
date:       2018-01-28
author:     振航
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - mysql
    - index
---
## 一、索引覆盖的概念

### 1.1 理解方式一
索引是高效找到行的一个方法，但是一般数据库也能使用索引找到一个列的数据，因此它不必读取整个行。毕竟索引叶子节点存储了它们索引的数据；当能通过读取索引就可以得到想要的数据，那就不需要读取行了。一个索引包含了（或覆盖了）满足查询结果的数据就叫做覆盖索引。 

### 1.2 理解方式二
是非聚集复合索引的一种形式，它包括在查询里的Select、Join和Where子句用到的所有列（即建索引的字段正好是覆盖查询条件中所涉及的字段，也即，索引包含了查询正在查找的数据）。

## 二、效果
减少一次拿实际数据的IO，此IO最为耗时。 

## 三、分页优化技巧
### 3.1 sql对比
```mysql
explain select * from sh_housedel_log a join (SELECT id FROM sh_housedel_log order by business_id limit 10000000,50) as b on a.id=b.id;

explain SELECT * FROM sh_housedel_log  order by business_id limit 10000000,50;
```
bussiness_id 有索引，id为主键。

### 3.2 分析

#### 3.2.1 sql1的方式
通过索引覆盖（返回主键索引，条件中有bussiness_id，Innodb的辅助索引叶子节点包含的是主键列，所以主键一定是被索引覆盖的），通过几次IO扫描辅助索引树即可，无需查出具体数据。
这样limit操作的成本也较低，limit 的1000000010条数据（此数据只包含id），得到最终10条数据，再通过ID映射。

#### 3.2.2 sql2的方式
limit 1000000000，10，则需要加载1000000010全量数据，然后扔掉1000000000条全量，成本很高。

## 四、其他方法
无条件，且主键id从0自增 , 则推荐用where条件过滤 ( 加载数据用 )
```mysql
SELECT * FROM sh_housedel_log where id > 10000000 order by id asc limit 50;
```
## 五、何为延迟关联
延迟关联主要体现为两点：
- 使用join ... （关联）
- 查到具体ID后才会执行关联 （延迟）

## 六、limit 和 order by

### 6.1 limit
正常情况下全表扫描，索引覆盖的情况走走索引。

### 6.1 order by
[参考](http://www.zuimoban.com/jiaocheng/mysql/8946.html)
