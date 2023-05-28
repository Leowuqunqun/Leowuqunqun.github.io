---
title: "深入学习MySQL事务：ACID特性的实现原理"
date: 2019-10-20
draft: false
layout: posts
tags: ["数据库","MySQL","事务"]
cover: 
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271109496.png"
---

# 问题
## Q1：什么是数据库事务

- 简单来说，事务就是要保证一组数据库操作，要么全部成功，要么全部失败
## Q2：为什么现在大部分都使用InnoDB引擎，而不是MyISAM？

- 其中原因之一是MyISAM不支持事务

# ACID 事务 
## 事务的四大特性
### 原子性
语句要么全执行，要么全不执行，是事务最核心的特性，事务本身就是以原子性来定义的；实现主要基于undo log
### 一致性

事务追求的最终目标，一致性的实现既需要数据库层面的保障，也需要应用层面的保障
### 隔离性
保证事务执行尽可能不受其他事务影响；InnoDB默认的隔离级别是RR，RR的实现主要基于锁机制（包含next-key lock）、MVCC（包括数据的隐藏列、基于undo log的版本链、ReadView）
### 持久性
保证事务提交后不会因为宕机等原因导致数据丢失；实现主要基于redo log

## 事务的四种隔离级别
### 读未提交 （Read uncommitted）
别人改数据的事务尚未提交，我在我的事务中也能读到。
### 读已提交（Read committed）
别人改数据的事务已经提交，我在我的事务中才能读到。
### 可重复读（**Repeatable read**）
别人改数据的事务已经提交，我在我的事务中也不去读。
### 串行化
我的事务尚未提交，别人就别想改数据。

最后
这4种隔离级别，并行性能依次降低，安全性依次提高。

## 并发事务带来的问题 
### 脏读
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271109496.png)
### 幻读
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271110032.png)
### 不可重复度
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271110667.png)
### 数据丢失
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271110194.png)


# 参考
[https://www.cnblogs.com/kismetv/p/10331633.html](https://www.cnblogs.com/kismetv/p/10331633.html)



