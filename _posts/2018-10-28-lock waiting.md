---
layout:     post
title:      lock wait timeout exceeded; try restarting transaction
subtitle:   锁等待
date:       2018-10-28
author:     振航
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - mysql
    - lock
---
## 问题分析
锁等待,基本上都是大事物未提交，阻塞其他事物拿锁。

## 排查方式
看事务表INNODB_TRX，里面是否有正在锁定的事务线程，看看ID是否在show processlist里面的sleep线程中，如果是，就证明这个sleep的线程事务一直没有commit或者rollback而是卡住了，我们需要手动kill掉.

## 开发环境最快的解决方式：
将当前的sleep事物线程全删掉,这样会导致删掉正常的事物信息（视具体情况而定,不要瞎比搞）。
```mysql
# 查看sleep线程信息
select * from information_schema.processlist  where COMMAND='Sleep'; 

# 获取干掉sleep线程的kill执行语句
select concat('KILL ',id,';') from information_schema.processlist where COMMAND='Sleep';;

# 执行上一步获取到的kill语句;
```

   


   



