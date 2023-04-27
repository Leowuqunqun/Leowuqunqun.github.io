---
title: "Kafka Rebalance机制"
date: 2020-08-01
draft: false
layout: posts
---

# 什么是Rebalance？
Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 consumer 如何达成一致，来分配订阅 Topic 的每个分区。
# 触发 Rebalance 的时机

- 组成员个数发生变化。例如有新的 consumer 实例加入该消费组或者离开组。
- 订阅的 Topic 个数发生变化。
- 订阅 Topic 的分区数发生变化。
# Rebalance流程
重平衡的完整流程需要消费者端和协调者组件共同参与才能完成。我们先从消费者的视角来审视一下重平衡的流程。
在消费者端，重平衡分为两个步骤：分别是加入组和等待领导消费者（Leader Consumer）分配方案。这两个步骤分别对应两类特定的请求：**JoinGroup 请求和 SyncGroup 请求。**

## JoinGroup

JoinGroup 请求的主要作用是将组成员订阅信息发送给领导者消费者，待领导者制定好分配方案后，重平衡流程进入到 SyncGroup 请求阶段。

![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1656395933559-acc05ca8-23da-42d1-a1cd-96ec41a6e24d.png)

## SyncGroup

SyncGroup 请求的主要目的，就是让协调者把领导者制定的分配方案下发给各个组内成员。当所有成员都成功接收到分配方案后，消费者组进入到 Stable 状态，即开始正常的消费工作。

![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1656395950438-cb024ff4-1e55-4167-82dd-ce074f6003d5.png)
# Rebalance问题处理思路
上面说到rebalance的三种情况，有一种情况是组成员崩溃是需要我们预防的
需要先搞清楚 kafaka 消费者配置的四个参数：

- **session.timeout.ms **： consumer 向 broker 发送心跳的超时时间。例如 session.timeout.ms = 180000 表示在最长 180 秒内 broker 没收到 consumer 的心跳，那么 broker 就认为该 consumer 死亡了，会启动 rebalance。
- **heartbeat.interval.ms** ：consumer 每次向 broker 发送心跳的时间间隔。heartbeat.interval.ms = 60000 表示 consumer 每 60 秒向 broker 发送一次心跳。一般来说，session.timeout.ms 的值是 heartbeat.interval.ms 值的 3 倍以上。
- **max.poll.interval.ms** ：consumer 每两次 poll 消息的时间间隔。简单地说，其实就是 consumer 每次消费消息的时长。如果消息处理的逻辑很重，那么市场就要相应延长。否则如果时间到了 consumer 还么消费完，broker 会默认认为 consumer 死了，发起 rebalance。
- **max.poll.records **：每次消费的时候，获取多少条消息。获取的消息条数越多，需要处理的时间越长。所以每次拉取的消息数不能太多，需要保证在 max.poll.interval.ms 设置的时间内能消费完，否则会发生 rebalance。

**消费者心跳超时，导致 rebalance。**
**消费者处理时间过长，导致 rebalance。**
**合理设置这些值，能避免非主动rebalance的次数**
# Rebalance带来的额外问题
在 Rebalance 的过程中 consumer group 下的所有消费者实例都会**停止工作**，等待 Rebalance 过程完成。
