---
title: "数据同步组件之Canal"
date: 2020-09-18T18:23:35+08:00
draft: false
tags: ["数据同步","异构数据","Canal"]
layout: posts
---

# 什么是Canal？
主要用途是基于 **MySQL 数据库增量日志解析**，提供**增量数据订阅和消费**。
那么各大厂商也是有很多基于MySQL进行扩展的数据库也是支持的，类似TiDB等
# Canal架构
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1656900021519-45d311fb-66f8-43a9-8392-3754cfaa0d50.png)
server代表一个canal运行实例，对应于一个jvm
instance对应于一个数据队列
instance模块：
eventParser (数据源接入，模拟slave协议和master进行交互，协议解析)
eventSink (Parser和Store链接器，进行数据过滤，加工，分发的工作)
eventStore (数据存储)
metaManager (增量订阅&消费信息管理器)

# Canal工作原理

#### MySQL主备复制原理

![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202304221126760.png)

- MySQL master 将数据变更写入二进制日志( binary log, 其中记录叫做二进制日志事件binary log events，可以通过 show binlog events 进行查看)
- MySQL slave 将 master 的 binary log events 拷贝到它的中继日志(relay log)
- MySQL slave 重放 relay log 中事件，将数据变更反映它自己的数据
#### Canal 工作原理

- Canal 模拟 MySQL slave 的交互协议，伪装自己为 MySQL slave ，向 MySQL master 发送dump 协议
- MySQL master 收到 dump 请求，开始推送 binary log 给 slave (即 Canal )
- Canal 解析 binary log 对象(原始为 byte 流)

# 应用场景

## 数据异构
在大型网站架构中，DB都会采用分库分表来解决容量和性能问题，但分库分表之后带来的新问题。比如不同维度的查询或者聚合查询，此时就会非常棘手。一般我们会通过数据异构机制来解决此问题。
所谓的数据异构，那就是将需要join查询的多表按照某一个维度又聚合在一个DB中。让你去查询。canal就是实现数据异构的手段之一。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1656900503810-84ee23b0-3fbb-4185-af7b-7c551e8f3797.png)

## 下发任务
另一种常见应用场景是下发任务，当数据变更时需要通知其他依赖系统。其原理是任务系统监听数据库变更，然后将变更的数据写入MQ/kafka进行任务下发，比如商品数据变更后需要通知商品详情页、列表页、搜索页等先关系统。这种方式可以保证数据下发的精确性，通过MQ发送消息通知变更缓存是无法做到这一点的，而且业务系统中不会散落着各种下发MQ的代码，从而实现了下发归集，如下图所示。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1656900486593-9d278e71-d6fd-4f9f-b878-0345509ea5e8.png)

# 其他数据源
MongoDB参考 MongoShake
