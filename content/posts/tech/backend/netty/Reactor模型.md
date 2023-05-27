---
title: "Reactor模型"
date: 2019-03-04
draft: false
layout: posts
tags: ["Java","Netty","Reactor模型"]
---

# Reactor
## 模型定义的三个角色

- Reactor：负责监听和分配事件，将I/O事件分派给对应的Handler。新的事件包含连接建立就绪、读就绪、写就绪等。
- Acceptor：处理客户端新连接，并分派请求到处理器链中。
- Handler：将自身与事件绑定，执行非阻塞读/写任务，完成channel的读入，完成处理业务逻辑后，负责将结果写出channel。可用资源池来管理。
## 处理请求流程
读取操作：

1. 应用程序注册读就绪事件和相关联的事件处理器
2. 事件分离器等待事件的发生
3. 当发生读就绪事件的时候，事件分离器调用第一步注册的事件处理器
## 主从Reactor多线程模型

1. mainReactor负责监听server socket，用来处理网络IO连接建立操作，将建立的socketChannel指定注册给subReactor。
2. subReactor主要做和建立起来的socket做数据交互和事件业务处理操作。通常，subReactor个数上可与CPU个数等同。

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271007544.png)
消息处理流程：

1. 从主线程池中随机选择一个Reactor线程作为acceptor线程，用于绑定监听端口，接收客户端连接
2. acceptor线程接收客户端连接请求之后创建新的SocketChannel，将其注册到主线程池的其它Reactor线程上，由其负责接入认证、IP黑白名单过滤、握手等操作
3. 步骤2完成之后，业务层的链路正式建立，将SocketChannel从主线程池的Reactor线程的多路复用器上摘除，重新注册到Sub线程池的线程上，并创建一个Handler用于处理各种连接事件
4. 当有新的事件发生时，SubReactor会调用连接对应的Handler进行响应
5. Handler通过Read读取数据后，会分发给后面的Worker线程池进行业务处理
6. Worker线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给Handler进行处理
7. Handler收到响应结果后通过Send将响应结果返回给Client

# 总结
Reactor模型具有如下的优点：

1. 响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；
2. 编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；
3. 可扩展性，可以方便地通过增加Reactor实例个数来充分利用CPU资源；
4. 可复用性，Reactor模型本身与具体事件处理逻辑无关，具有很高的复用性。
