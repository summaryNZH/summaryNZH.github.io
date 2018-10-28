---
layout:     post
title:      concurrent duplicate insert dead lock
subtitle:   
date:       2018-01-28
author:     振航
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - mysql
    - lock
---
## 一、遇到的问题
并行执行重复的insert出现了死锁。然而死锁是怎么产生的呢？

## 二、dead lock
死锁是由于多个事务相互持有其他事务所需要的锁，结果导致事务都无法继续，进而触发死锁检测，
其中某个事务会被回滚，释放相应的锁，其他事务得以正常继续；简言之，就是多个事务之间的锁等待产生了回路，死循环了；
死锁发生时，会立刻被检测到，并且回滚其中某个事务，而不会长时间阻塞、等待；

## 三、mysql doc
> If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session already has an exclusive lock. This can occur if another session deletes the row.

##### 翻译
如果出现重复键错误，将给重复索引记录加上共享锁（s）。如果有多个会话试图插入同一行(如果另一个会话已经具有独占锁（x锁）)，则使用共享锁可能导致死锁。如果另一个会话删除行，或者insert操作事物回滚，就会发生这种情况。

## 四、插入加锁过程
INSERT 语句将在插入的记录上加一把排他锁，这个锁是一个index-record lock，并不是next-key 锁，因此就没有gap 锁，他将不会阻止其他会话在该条记录之前的gap插入记录。

在插入记录之前，将会加上一种叫做 insert intention gap 的 gap 锁。这个 insert intention gap表示他有意向在这个index gap插入记录，如果其他会话在这个index gap中插入的位置不相同，那么将不需要等待。假设存在索引记录4和7，会话A要插入记录5，会话B要插入记录6，每个会话在插入记录之前都需要锁定4和7之间gap，但是他们彼此不会互相堵塞，因为插入的位置不相同。

如果出现了重复键错误，将在重复键上加一个共享锁。如果会话1插入一条记录，没有提交，他会在该记录上加上排他锁，会话2和会话3都尝试插入该重复记录，那么他们都会被堵塞，会话2和会话3将尝试在该记录上加一个共享锁。如果此时会话1回滚，将发生死锁。

## 具体案例
### 表结构
```mysql
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
```
session1
```mysql
START TRANSACTION; INSERT INTO t1 VALUES(1);
```
session2
```mysql
START TRANSACTION; INSERT INTO t1 VALUES(1);

```
session3
```mysql
START TRANSACTION; INSERT INTO t1 VALUES(1);
```
session1
```mysql
ROLLBACK;
```
为什么会发生死锁呢？当会话1进行回滚的时候，记录上的排他锁释放了，会话2和会话3都获得了共享锁。然后会话2和会话3都想要获得排他锁，进而发生了死锁。

还有一个类似的例子

session1
```mysql
START TRANSACTION; DELETE FROM t1 WHERE i = 1;
```
session2
```mysql
START TRANSACTION; INSERT INTO t1 VALUES(1);
```
session3
```mysql
START TRANSACTION; INSERT INTO t1 VALUES(1);
```
session1
```mysql
COMMIT;
```

会话1在该记录上拥有一把排他锁，会话2和会话3都碰到了重复记录，因此都在申请共享锁。当会话1提交之后，会话1释放了排他锁，之后的会话2会话3先后获得了共享锁，此时他们发生了死锁，因为会话2和会话3都无法或者排他锁，因为彼此都占用了该记录的共享锁。

## 五、总结
我理解 简单说就是都session 2 和 3 分别加了s锁后， 都在死等x锁，而2 和 3 分别都只能在死锁的情况下 才会释放s锁。因此死锁了。