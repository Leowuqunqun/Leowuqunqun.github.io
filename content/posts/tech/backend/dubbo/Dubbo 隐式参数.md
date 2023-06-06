---
title: "Dubbo 隐式参数解析"
date: 2020-03-18
draft: false
layout: posts
tags: ["dubbo"]
---



Dubbo 的隐式参数是通过 `RpcContext` 类来实现的。在 Dubbo 中，每个服务调用都与一个 `RpcContext` 对象相关联，该对象通常被绑定到当前线程上，并在服务调用过程中用于传递上下文参数和设置响应结果。你可以通过调用 `RpcContext.getContext()` 方法来访问当前服务调用的上下文。

下面是 Dubbo 隐式参数相关代码的源码解析：

### 服务消费者

在服务消费者中，通过设置 `referenceConfig.setShareContext(true)` 来启用隐式传递功能。在服务消费者调用远程方法之前，需要将需要传递的参数设置到 `RpcContext` 中。例如，下面的代码将 `index` 参数设置为 `1`：

```java
RpcContext.getContext().setAttachment("index", "1");
```

在调用远程方法时，Dubbo 会自动将 `RpcContext` 中的参数传递给服务提供方，你可以通过 `RpcContext` 的 `getUrl()` 方法来获取引用服务的 URL：

```java
DemoService demoService = referenceConfig.get();
String result = demoService.sayHello("world");
URL serviceUrl = RpcContext.getContext().getUrl();
```

### 服务提供者

在服务提供者中，也可以通过 `RpcContext` 来获取调用的上下文信息。例如，下面的代码从 `RpcContext` 中获取 `index` 参数：

```java
String index = RpcContext.getContext().getAttachment("index");
```

服务提供者也可以通过 `RpcContext` 设置响应结果。例如，下面的代码将 `response` 参数设置到 `RpcContext` 中：

```java
RpcContext.getContext().setResponse(response);
```

在服务调用完成后，Dubbo 会自动将 `RpcContext` 中的信息传递回服务消费者。

### 隐式参数源码解析

在 Dubbo 中，隐式参数主要使用 `RpcContext` 类来实现。`RpcContext` 类实现了 `Threadlocal` 模式，用于在线程内共享调用上下文信息。每个线程都会有一个 `ThreadLocal` 实例，用于存储当前线程的 `RpcContext` 对象。Dubbo 在服务调用过程中，会将当前线程的 `RpcContext` 对象绑定到 `ThreadLocal` 中，以便在后续处理过程中方便访问和传递。

Dubbo 的隐式传递功能的实现主要依赖于 `InvokerInvocationHandler` 类和 `RpcInvocationFilter` 拦截器。在服务消费者发送请求之前，Dubbo 会通过 `InvokerInvocationHandler` 将 `RpcContext` 中的信息合并到 `RpcInvocation` 对象中，并将最终的调用参数传递给服务提供者。在服务提供者接收到请求之后，Dubbo 会使用 `RpcInvocationFilter` 拦截器来将 `RpcInvocation` 中的信息还原到 `RpcContext` 中，并执行响应结果的设置。

总之，在 Dubbo 中，隐式参数通过 `RpcContext` 类实现，其核心思想是在线程内共享调用上下文信息，通过 `ThreadLocal` 实现线程安全。在 Dubbo 实现隐式传递的过程中，服务消费者和服务提供者之间需要通过 `RpcInvocation` 对象传递调用参数，从而实现上下文信息的传递和响应结果的返回。

### 应用场景

* 链路追踪 提供TraceId的透传
* 传递访问用户的身份认证的上下文，jwt，用户信息等
* 想自定义一些信息传递，偏公共基础的内容

### 使用注意

#### Dubbo 2.6.x

RpcContext类就是Dubbo的上下文，但是它仅仅是一个ThreadLocal级别的**临时状态记录器**，当接收到RPC请求或发起RPC请求时，RpcContext的状态都会变化。比如：A调用B、B再调用C的情况下。**B机器**的RpcContext会有如下的情况发生：在B调用C之前，RpcContext记录的是A调B的信息，在B调用C之后，RpcContext记录的是B调C的信息。

<font color=red>会有参数丢失的问题，就是因为多重服务调用的原因</font>



#### Dubbo 3.x 

RpcContext 被拆分为四大模块（ServerContext、ClientAttachment、ServerAttachment 和 ServiceContext）。

- ServiceContext：在 Dubbo 内部使用，用于传递调用链路上的参数信息，如 invoker 对象等
- ClientAttachment：在 Client 端使用，往 ClientAttachment 中写入的参数将被传递到 Server 端
- ServerAttachment：在 Server 端使用，从 ServerAttachment 中读取的参数是从 Client 中传递过来的
- ServerContext：在 Client 端和 Server 端使用，用于从 Server 端回传 Client 端使用，Server 端写入到 ServerContext 的参数在调用结束后可以在 Client 端的 ServerContext 获取到

在 Dubbo 2.7 中，在 A 端设置的参数，调用 B 以后，如果 B 继续调用了 C，原来在 A 中设置的参数也会被带到 C 端过去，造成参数污染的问题。 Dubbo 3 对 RpcContext 进行了重构，支持可选参数透传，。默认开启参数透传