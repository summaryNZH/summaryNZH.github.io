---
layout:     post
title:      innodb sql lock
subtitle:   DML 加锁分析
date:       2018-01-28
author:     振航
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - mysql
    - lock
---
# DML 加锁分析

## 1.SELECT ... FROM 
是一个快照读，通过读取数据库的一个快照，不会加任何锁，除非将隔离级别设置成了 SERIALIZABLE 。在 SERIALIZABLE 隔离级别下，如果索引是非唯一索引，那么将在相应的记录上加上一个共享的next key锁。如果是唯一索引，只需要在相应记录上加index record lock。
## 2. SELECT ... FROM ... LOCK IN SHARE MODE 
在所有索引扫描范围的索引记录上加上共享的next key锁。如果是唯一索引，只需要在相应记录上加index record lock。
## 3. SELECT ... FROM ... FOR UPDATE 
在所有索引扫描范围的索引记录上加上排他的next key锁。如果是唯一索引，只需要在相应记录上加index record lock。这将堵塞其他会话利用SELECT ... FROM ... LOCK IN SHARE MODE 读取相同的记录，但是快照读将忽略记录上的锁。
## 4. UPDATE ... WHERE ...
在所有索引扫描范围的索引记录上加上排他的next key锁。如果是唯一索引，只需要在相应记录上加index record lock。
当UPDATE 操作修改主键记录的时候，将在相应的二级索引上加上隐式的锁。当进行重复键检测的时候，将会在插入新的二级索引记录之前，在其二级索引上加上一把共享锁。
## 5. DELETE FROM ... WHERE ... 
在所有索引扫描范围的索引记录上加上排他的next key锁。如果是唯一索引，只需要在相应记录上加index record lock。

## 6. INSERT 
将在插入的记录上加一把排他锁，这个锁是一个index-record lock，并不是next-key 锁，因此就没有gap 锁，他将不会阻止其他会话在该条记录之前的gap插入记录。
在插入记录之前，将会加上一种叫做 insert intention gap 的 gap 锁。这个 insert intention gap表示他有意向在这个index gap插入记录，如果其他会话在这个index gap中插入的位置不相同，
那么将不需要等待。假设存在索引记录4和7，会话A要插入记录5，会话B要插入记录6，每个会话在插入记录之前都需要锁定4和7之间gap，但是他们彼此不会互相堵塞，因为插入的位置不相同。
如果出现了重复键错误，将在重复键上加一个共享锁。如果会话1插入一条记录，没有提交，他会在该记录上加上排他锁，会话2和会话3都尝试插入该重复记录，那么他们都会被堵塞，
会话2和会话3将尝试在该记录上加一个共享锁。如果此时会话1回滚，将发生死锁(在rr隔离级别下还是会加next key的也就是说你不要被这个误导,这个只是说insert做了什么,而没说在rr级别下做了什么)
## 7. INSERT ... ON DUPLICATE KEY UPDATE 
 和普通的INSERT并不相同。如果碰到重复键值，INSERT ... ON DUPLICATE KEY UPDATE 将在记录上加排他的 next-key锁。
## 8. REPLACE 
在没有碰到重复键值的时候和普通的INSERT是一样的，如果碰到重复键，将在记录上加一个排他的 next-key锁。
## 9. INSERT INTO T SELECT ... FROM S WHERE ... 
在插入T表的每条记录上加上 index record lock 。如果隔离级别是 READ COMMITTED, 或者启用了 innodb_locks_unsafe_for_binlog 且事务隔离级别不是SERIALIZABLE，那么innodb将通过快照读取表S(no locks)。否则，innodb将在S的记录上加共享的next-key锁。
CREATE TABLE ... SELECT ... 和 INSERT INTO T SELECT ... FROM S WHERE ... 一样，在S上加共享的next-key锁或者进行快照读取（(no locks） 
## 10. REPLACE INTO t SELECT ... FROM s WHERE ... 和 UPDATE t ... WHERE col IN (SELECT ... FROM s ...) 
REPLACE INTO t SELECT ... FROM s WHERE ... 和 UPDATE t ... WHERE col IN (SELECT ... FROM s ...) 中的select 部分将在表s上加共享的next-key锁。
## 11.AUTO-INC table lock
当碰到有自增列的表的时候，innodb在自增列的索引最后面加上一个排他锁，叫AUTO-INC table lock。AUTO-INC table lock会在语句执行完成后进行释放，而不是事务结束。如果AUTO-INC table lock被一个会话占有，那么其他会话将无法在该表中插入数据。innodb可以预先获取sql需要多少自增的大小，而不需要去申请锁，更多设置请参考参数innodb_autoinc_lock_mode.
## 12. 外键 插入更新删除
如果一张表的外键约束被启用了，任何在该表上的插入、更新、删除都将需要加共享的record-level locks来检查是否满足约束。如果约束检查失败，innodb也会加上共享的record-level locks。
## 13. lock tables
lock tables 是用来加表级锁，但是是MySQL的server层来加这把锁的。当innodb_table_locks = 1 (the default) 以及 autocommit = 0的时候，innodb能够感知表锁，同时server层了解到innodb已经加了row-level locks。否则，innodb将无法自动检测到死锁，同时server无法确定是否有行级锁，导致当其他会话占用行级锁的时候还能获得表锁。
## 14.快照读特性
> A consistent read means that InnoDB uses multi-versioning to present to a query a snapshot of the database at a point in time. The query sees the changes made by transactions that committed before that point of time, and no changes made by later or uncommitted transactions. The exception to this rule is that the query sees the changes made by earlier statements within the same transaction. This exception causes the following anomaly: If you update some rows in a table, a SELECT sees the latest version of the updated rows, but it might also see older versions of any rows. If other sessions simultaneously update the same table, the anomaly means that you might see the table in a state that never existed in the database. 

默认情况下，innodb使用MVCC进行快照读。查询只能查看到之前提交的事务更改的数据，之后提交的数据或者没有提交的事务数据是没办法看到的。但是这里有个例外就是同一个事务的查询能看到同一个事务里面的更改。因此当SESS1 在进行UPDATE的时候，会进行当前读，也就是读取所有已经提交的数据，相当于读取的是select * from t3 where where name ='a' for update;的结果集，然后进行UPDATE。之后的select * from t3 where where name ='a'看到的就是当前UPDATE之后的结果了。


