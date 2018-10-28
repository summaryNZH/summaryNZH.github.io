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

### 介绍：
- 用于插入时自增字段控制加锁的配置。

### 优点：
- 它的设置可以达到性能与安全(主从的数据一致性)的平衡
- 比如 配置auto_increment参数的字段，mysql可以保证这个字段在多进程操作时的原子性。就是通过这个innodb_autoinc_lock_mode 的配置来控制保证的类型级别。

### 参数配置

| 类型值       | 简介   |  原理  |   优点 | 缺点
| :-------:   | :-----:  | :----:  | :----: |:----: | 
| 0       | tradition（传统）   |每次写入操作都会生成 table-level auto-increment lock（以下简写，称为IALML）       | 绝对安全| 写入性能太差|
| 1(缺省值) |   consecutive（连续）  | bulk-insert的时候，还是会产生IALML，这个锁会在语句结束时释放，而不是一次事物，事物会包含多个语句，产生这个锁的原因，因为这个级别的insert不确定立刻确定插入的行数，因此要加上IALML锁来保证bulk-insert的时候，auto_increment的值不会别的轻量级锁获取。simple-insert的时候，会产生一个轻量锁，获取到auto_increment属性后就释放，如果上一个IALML没释放，会等待。 | 非常安全，性能非常优于=0| 在一些情况下还是会产生表级别的IALML|
| 2        |    interleaved（交错）  | 当进行bulk insert的时候，不会产生table级别的自增锁，因为它是允许其他insert插入的    | 性能最好，支持并发|SBR下不安全（复制出错不一致）(简单说复制后的sql执行顺序不一致，就会出现问题),一条bulk-insert语句得到的id可能不连续（结合配置为2 的解释很好理解） 1|



 

 
