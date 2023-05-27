---
title: "操作系统之CPU多级缓存"
date: 2014-03-05
draft: false
layout: posts
tags: ["操作系统"]
cover:
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271343980.png"
---

# 什么是CPU多级缓存？
## CPU缓存的来历
因为CPU和内存之间的频繁交互，内存的效率提升远不如CPU，**为了解决CPU运算速度与内存读写速度不匹配的矛盾**，就出现了CPU缓存。
## CPU缓存的概念
**CPU缓存是位于CPU与内存之间的临时数据交换器，它的容量比内存小的多但是交换速度却比内存要快得多。CPU缓存一般直接跟CPU芯片集成或位于主板总线互连的独立芯片上**。
为了简化与内存之间的通信，高速缓存控制器是针对数据块，而不是字节进行操作的。高速缓存其实就是一组称之为**缓存行**(Cache Line)的固定大小的数据块组成的，典型的一行是64字节，一般是根据系统来决定。
## CPU缓存的意义
CPU往往需要重复处理相同的数据、重复执行相同的指令，如果这部分数据、指令CPU能在CPU缓存中找到，CPU就不需要从内存或硬盘中再读取数据、指令，从而减少了整机的响应时间

- **时间局部性（Temporal Locality）**：如果一个信息项正在被访问，那么在近期它很可能还会被再次访问。
- **空间局部性（Spatial Locality）**：如果一个存储器的位置被引用，那么将来他附近的位置也会被引用。
# CPU的多级缓存
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271343980.png)越靠近 CPU 的缓存越快也越小。
所以 L1 缓存很小但很快，并且紧靠着在使用它的 CPU 内核。
L2 大一些，也慢一些，并且仍然只能被一个单独的 CPU 核使用。
L3 在现代多核机器中更普遍，仍然更大，更慢，并且被单个插槽上的所有 CPU 核共享。
最后，主存保存着程序运行的所有数据，它更大，更慢，由全部插槽上的所有 CPU 核共享。

## 缓存执行流程
当 CPU 执行运算的时候，它先去 L1 查找所需的数据，再去 L2，然后是 L3，最后如果这些缓存中都没有，所需的数据就要去主内存拿。
走得越远，运算耗费的时间就越长。
## CPU缓存一致性协议(MESI)
为了保证多个CPU缓存中共享数据的一致性，定义了缓存行(Cache Line)的四种状态，而CPU对缓存行的四种操作可能会产生不一致的状态，因此缓存控制器监听到本地操作和远程操作的时候，需要对地址一致的缓存行的状态进行一致性修改，从而保证数据在多个缓存之间保持一致性。

CPU中每个缓存行（Caceh line)使用4种状态进行标记，使用2bit来表示:

| 状态 | 描述 | 监听任务 | 状态转换 |
| --- | --- | --- | --- |
| M 修改 (Modified) | 该Cache line有效，数据被修改了，和内存中的数据不一致，数据只存在于本Cache中。 | 缓存行必须时刻监听所有试图读该缓存行相对就主存的操作，这种操作必须在缓存将该缓存行写回主存并将状态变成S（共享）状态之前被延迟执行。 | 当被写回主存之后，该缓存行的状态会变成独享（exclusive)状态。 |
| E 独享、互斥 (Exclusive) | 该Cache line有效，数据和内存中的数据一致，数据只存在于本Cache中。 | 缓存行也必须监听其它缓存读主存中该缓存行的操作，一旦有这种操作，该缓存行需要变成S（共享）状态。 | 当CPU修改该缓存行中内容时，该状态可以变成Modified状态 |
| S 共享 (Shared) | 该Cache line有效，数据和内存中的数据一致，数据存在于很多Cache中。 | 缓存行也必须监听其它缓存使该缓存行无效或者独享该缓存行的请求，并将该缓存行变成无效（Invalid）。 | 当有一个CPU修改该缓存行时，其它CPU中该缓存行可以被作废（变成无效状态 Invalid）。 |
| I 无效 (Invalid) | 该Cache line无效。 | 无 | 无 |

> **注意**：
**对于M和E状态而言总是精确的，他们在和该缓存行的真正状态是一致的，而S状态可能是非一致的**。如果一个缓存将处于S状态的缓存行作废了，而另一个缓存实际上可能已经独享了该缓存行，但是该缓存却不会将该缓存行升迁为E状态，这是因为其它缓存不会广播他们作废掉该缓存行的通知，同样由于缓存并没有保存该缓存行的copy的数量，因此（即使有这种通知）也没有办法确定自己是否已经独享了该缓存行。

从上面的意义看来E状态是一种投机性的优化：如果一个CPU想修改一个处于S状态的缓存行，总线事务需要将所有该缓存行的copy变成invalid状态，而修改E状态的缓存不需要使用总线事务。
MESI状态转换图：
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271343196.png)

## 多核缓存协同操作
#### (1) 内存变量
假设有三个CPU A、B、C，对应三个缓存分别是cache a、b、c。在主内存中定义了x的引用值为0。
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271343091.png)
**内存变量**

#### (2) 单核读取
执行流程是：

- CPU A发出了一条指令，从主内存中读取x。
- 从主内存通过 bus 读取到 CPU A 的缓存中（远端读取 Remote read）,这时该 Cache line 修改为 E 状态（独享）。

![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271343326.png)
**单核读取**

#### (3) 双核读取
执行流程是：

- CPU A发出了一条指令，从主内存中读取x。
- CPU A从主内存通过bus读取到 cache a 中并将该 Cache line 设置为E状态。
- CPU B发出了一条指令，从主内存中读取x。
- CPU B试图从主内存中读取x时，CPU A检测到了地址冲突。这时CPU A对相关数据做出响应。此时x存储于 cache a 和 cache b 中，x在 chche a 和 cache b 中都被设置为S状态(共享)。

![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271343113.png)
**双核读取**

#### (4) 修改数据
执行流程是：

- CPU A 计算完成后发指令需要修改x.
- CPU A 将x设置为M状态（修改）并通知缓存了x的 CPU B, CPU B 将本地 cache b 中的x设置为I状态(无效)
- CPU A 对x进行赋值。

![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271344573.png)
**修改数据**

#### (5) 同步数据
那么执行流程是：

- CPU B 发出了要读取x的指令。
- CPU B 通知CPU A,CPU A将修改后的数据同步到主内存时cache a 修改为E（独享）
- CPU A同步CPU B的x,将cache a和同步后cache b中的x设置为S状态（共享）。

![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271344868.png)
**同步数据**



