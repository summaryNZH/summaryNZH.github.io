---
layout:     post
title:      Auto-Inc locks
subtitle:   mysql 自增锁
date:       2018-10-26
author:     振航
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - mysql
    - lock
---

# Mysql Auto-Inc locks

## The types of insert

### insert like
> 任何插入操作只要产生了新的记录都叫insert-like,包括 simple insert and bulk insert and mixed insert.
```mysql
insert，insert ... select，replace, replace ... select, load data
```
### simple-insert
> 插入的行数是固定的, 但不包括：insert...on duplicate key update
```mysql
insert into values，replace.
```
### bulk-inserts
> 插入的行数不能立刻确定
```mysql
insert...select , replace...select, load data
```  
### mixed-insert
> 自增列（auto_increment值不确定）或者 insert on duplicate key update
```mysql
insert ...on duplicate key update;
INSERT INTO oplog (id, log_id, field, field_desc, old_value, new_value) VALUES (36, null, '室数', '室数', '无', '3'); //log_id（auto_increment）
```  

## innodb_auto_inc_lock_mode 参数

#### 介绍：
用于插入时自增字段控制加锁的配置。
#### 优点：
它的设置可以达到性能与安全(主从的数据一致性)的平衡
>比如 配置auto_increment参数的字段，mysql可以保证这个字段在多进程操作时的原子性。就是通过这个innodb_autoinc_lock_mode 的配置来控制保证的类型级别。
#### 参数配置
[](https://github.com/summaryNZH/Java/blob/master/baseinfo/img/authinclock.png)



 

 
