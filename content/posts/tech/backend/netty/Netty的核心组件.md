---
title: "Netty的核心组件"
date: 2019-02-03
draft: false
layout: posts
tags: ["Java","Netty"]
---

# Channel
**Channel** 接口是 **Netty** 对网络操作抽象类，它除了包括基本的 **I/O** 操作，如 **bind()**、**connect()**、**read()**、**write()** 等。
比较常用的**Channel**接口实现类是**NioServerSocketChannel**（服务端）和**NioSocketChannel**（客户端），这两个 **Channel** 可以和 **BIO** 编程模型中的**ServerSocket**以及**Socket**两个概念对应上。**Netty** 的 **Channel** 接口所提供的 **API**，大大地降低了直接使用 **Socket** 类的复杂性。
# EventLoop
**EventLoop 的主要作用实际就是负责监听网络事件并调用事件处理器进行相关 I/O 操作的处理。**
那 **Channel** 和 **EventLoop** 直接有啥联系呢？
Channel 为 Netty 网络操作(读写等操作)抽象类，**EventLoop** 负责处理注册到其上的**Channel** 处理 I/O 操作，两者配合参与 I/O 操作。
# ChannelFuture
**Netty** 是异步非阻塞的，所有的 I/O 操作都为异步的。
因此，我们不能立刻得到操作是否执行成功，但是，你可以通过 **ChannelFuture** 接口的 **addListener()** 方法注册一个 **ChannelFutureListener**，当操作执行成功或者失败时，监听就会自动触发返回结果。

# ChannelHandler和ChannelPipeline
**ChannelHandler** 是消息的具体处理器。他负责处理读写操作、客户端连接等事情。
**ChannelPipeline** 为 **ChannelHandler** 的链，提供了一个容器并定义了用于沿着链传播入站和出站事件流的 API 。当 **Channel** 被创建时，它会被自动地分配到它专属的**ChannelPipeline**。
我们可以在 **ChannelPipeline** 上通过 **addLast()** 方法添加一个或者多个**ChannelHandler** ，因为一个数据或者事件可能会被多个 **Handler** 处理。当一个 **ChannelHandler** 处理完之后就将数据交给下一个 **ChannelHandler** 。
有点类似责任链
#  

