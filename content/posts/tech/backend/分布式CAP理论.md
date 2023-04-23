---
title: "分布式CAP理论"
date: 2019-09-16
draft: false
layout: posts
---

## CAP理论
CAP 理论对分布式系统的特性做了高度抽象，形成了三个指标

- 一致性（Consistency）
- 可用性（Availability）
- 分区容错性（Partition Tolerance）
### 一致性
一致性说的是客户端的每次读操作，不管访问哪个节点，要么读到的都是同一份最新写入的数据，要么读取失败。

比如，2 个节点的 KV 存储，原始的 KV 记录为“X = 1”。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915604125-b3746855-fd29-47f7-b69e-5a9714f5fa1e.png)
紧接着，客户端向节点 1 发送写请求“SET X = 2”。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915616165-9aa07806-1ae2-4787-9e43-2ca4a9d4644c.png)
如果节点 1 收到写请求后，只将节点 1 的 X 值更新为 2，然后返回成功给客户端。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915632234-c67c054f-f689-4e68-9846-4135cc3bd77f.png)
那么，此时如果客户端访问节点 2 执行读操作，就无法读到最新写入的 X 值，这就不满足一致性了。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915642322-b0210d64-56db-4f84-9bae-2993221ef065.png)
如果节点 1 收到写请求后，通过节点间的通讯，同时将节点 1 和节点 2 的 X 值都更新为 2，然后返回成功给客户端。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915659361-672a597a-9c14-4f66-99f9-70c071f0368b.png)
那么在完成写请求后，不管客户端访问哪个节点，读取到的都是同一份最新写入的数据，这就叫一致性。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915672422-14e3e3fe-88e7-41d9-9a6c-8e4ff882c5aa.png)
一致性这个指标，描述的是分布式系统非常重要的一个特性，强调的是数据正确。也就是说，对客户端而言，每次读都能读取到最新写入的数据。

### 可用性
**可用性说的是任何来自客户端的请求，不管访问哪个非故障节点，都能得到响应数据，但不保证是同一份最新数据。**
比如，用户可以选择向节点 1 或节点 2 发起读操作，如果不管节点间的数据是否一致，只要节点服务器收到请求，就响应 X 的值，那么，2 个节点的服务是满足可用性的。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915731629-7c340ca6-b9ff-476b-b782-1f1c077d1945.png)

### 分区容错性
分区容错性说的是当节点间出现任意数量的消息丢失或高延迟的时候，系统仍然在继续工作。也就是说，分布式系统在告诉访问本系统的客户端：不管我的内部出现什么样的数据同步问题，我会一直运行。这个指标，强调的是集群对分区故障的容错能力。
比如当节点 1 和节点 2 通信出问题的时候，如果系统仍能继续工作，那么，2 个节点是满足分区容错性的。
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915802989-2b9446f9-b1ed-4f93-b9ce-d40d2afcdf21.png)

**因为分布式系统与单机系统不同，它涉及到多节点间的通讯和交互，节点间的分区故障是必然发生的，所以我要提醒你的是，在分布式系统中分区容错性是必须要考虑的。**
## CAP不可能三角
CAP 不可能三角说的是对于一个分布式系统而言，一致性（Consistency）、可用性（Availability）、分区容错性（Partition Tolerance）3 个指标不可兼得，只能在 3 个指标中选择 2 个。
![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1651915879628-0ed7e0ee-7d8d-49f9-b37c-563a29ea35df.png)
**那么在分布式系统中，P是一定要保证的，所以我们一般是分区故障时在AC之间做出平衡。**

- 当选择了一致性（C）的时候，一定会读到最新的数据，不会读到旧数据，但如果因为消息丢失、延迟过高发生了网络分区，那么这个时候，当集群节点接收到来自客户端的读请求时，为了不破坏一致性，可能会因为无法响应最新数据，而返回出错信息。
- 当选择了可用性（A）的时候，系统将始终处理客户端的查询，返回特定信息，如果发生了网络分区，一些节点将无法返回最新的特定信息，它们将返回自己当前的相对新的信息。
## 如何决策CAP

- CA 模型，在分布式系统中不存在。因为舍弃 P，意味着舍弃分布式系统，就比如单机版关系型数据库 MySQL，如果 MySQL 要考虑主备或集群部署时，它必须考虑 P。
- CP 模型，采用 CP 模型的分布式系统，舍弃了可用性，一定会读到最新数据，不会读到旧数据。一旦因为消息丢失、延迟过高发生了网络分区，就影响用户的体验和业务的可用性（比如基于 Raft 的强一致性系统，此时可能无法执行读操作和写操作）。典型的应用是 Etcd，Consul 和 Hbase。
- AP 模型，采用 AP 模型的分布式系统，舍弃了一致性，实现了服务的高可用。用户访问系统的时候，都能得到响应数据，不会出现响应错误，但会读到旧数据。典型应用就比如 Cassandra 和 DynamoDB。
