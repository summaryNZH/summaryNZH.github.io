---
layout:     post
title:      network card overflow and redis timeout
subtitle:   网卡流量爆点-redis超时
date:       2018-01-28
author:     振航
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - redis
    - socket
---
## 一、网卡什么时候会被打爆?
网卡计量用的是b\bit,和我们通常计量的B\byte换算关系是8:1，所以100M的网卡跑12M左右达到最大值（其他类型网卡也是这个比例）。

查看linux下网络流量可以使用iftop ，可以精确的看到每个IP的流量。

## redis超时常见原因
![](https://raw.githubusercontent.com/summaryNZH/Java/master/baseinfo/img/redis.png)

1. 网卡被打满（上图可看出来是这种情况）
2. 有慢查询

## 单机qps -经验
- 简单的get、set、pipeline 而且key不大的情况下可以勉强抗12w。
- 如果没使用pipeline就是8W的qps，以上描述都为长连接情况下。

   

