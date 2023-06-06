---
title: "一文读懂Apollo核心（附实践使用注意事项）"
date: 2020-06-20
draft: false
layout: posts
tags: ["Apollo","配置中心"]
cover:
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/imageapollo%E6%9E%B6%E6%9E%84.png"
---

# 整体架构
![apollo架构](https://raw.githubusercontent.com/Leowuqunqun/img/master/imageapollo%E6%9E%B6%E6%9E%84.png)

## ConfigService

- 提供配置的读取、推送等功能，服务对象是Apollo客户端
## Admin Service

- 提供配置的修改、发布等功能，服务对象是Apollo Portal（管理界面）
## Eureka

- Config Service和Admin Service都是多实例、无状态部署，所以需要将自己注册到Eureka中并保持心跳
## Meta Server

- Meta Server从Eureka获取Config Service和Admin Service的服务信息，相当于是一个Eureka Client
## Portal

- 提供Web界面供用户管理配置
- 通过Meta Server获取Admin Service服务列表（IP+Port），通过IP+Port访问服务
- 在Portal侧做load balance、错误重试
## Client

- Apollo提供的客户端程序，为应用提供配置获取、实时更新等功能
- 通过Meta Server获取Config Service服务列表（IP+Port），通过IP+Port访问服务
- 在Client侧做load balance、错误重试

# 动态配置推送设计

![apollo动态配置](https://raw.githubusercontent.com/Leowuqunqun/img/master/imageapollo%E5%8A%A8%E6%80%81%E9%85%8D%E7%BD%AE.png)

1. 用户在Portal操作配置发布
2. Portal调用Admin Service的接口操作发布
3. Admin Service发布配置后，发送ReleaseMessage给各个Config Service
4. Config Service收到ReleaseMessage后，通知对应的客户端
### 发送ReleaseMessage的实现方式
Admin Service在配置发布后，需要通知所有的Config Service有配置发布，从而Config Service可以通知对应的客户端来拉取最新的配置。
从概念上来看，这是一个典型的消息使用场景，Admin Service作为producer发出消息，各个Config Service作为consumer消费消息。通过一个消息组件（Message Queue）就能很好的实现Admin Service和Config Service的解耦。
在实现上，考虑到Apollo的实际使用场景，以及为了尽可能减少外部依赖，我们没有采用外部的消息中间件，而是通过数据库实现了一个简单的消息队列。
实现方式如下：

1. Admin Service在配置发布后会往ReleaseMessage表插入一条消息记录，消息内容就是配置发布的AppId+Cluster+Namespace，参见[DatabaseMessageSender](https://github.com/apolloconfig/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/DatabaseMessageSender.java)
2. Config Service有一个线程会每秒扫描一次ReleaseMessage表，看看是否有新的消息记录，参见[ReleaseMessageScanner](https://github.com/apolloconfig/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageScanner.java)
3. Config Service如果发现有新的消息记录，那么就会通知到所有的消息监听器（[ReleaseMessageListener](https://github.com/apolloconfig/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageListener.java)），如[NotificationControllerV2](https://github.com/apolloconfig/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/controller/NotificationControllerV2.java)，消息监听器的注册过程参见[ConfigServiceAutoConfiguration](https://github.com/apolloconfig/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/ConfigServiceAutoConfiguration.java)
4. NotificationControllerV2得到配置发布的AppId+Cluster+Namespace后，会通知对应的客户端

示意图如下：
![apollo-ReleaseMessage](https://raw.githubusercontent.com/Leowuqunqun/img/master/imageapollo-ReleaseMessage.png)

### Config Service通知客户端的实现方式
刚才简要描述了NotificationControllerV2是如何得知有配置发布的，那NotificationControllerV2在得知有配置发布后是如何通知到客户端的呢？
实现方式如下：

1. 客户端会发起一个Http请求到Config Service的notifications/v2接口，也就是[NotificationControllerV2](https://github.com/apolloconfig/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/controller/NotificationControllerV2.java)，参见[RemoteConfigLongPollService](https://github.com/apolloconfig/apollo/blob/master/apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java)
2. NotificationControllerV2不会立即返回结果，而是通过[Spring DeferredResult](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html)把请求挂起
3. 如果在60秒内没有该客户端关心的配置发布，那么会返回Http状态码304给客户端
4. 如果有该客户端关心的配置发布，NotificationControllerV2会调用DeferredResult的[setResult](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html#setResult-T-)方法，传入有配置变化的namespace信息，同时该请求会立即返回。客户端从返回的结果中获取到配置变化的namespace后，会立即请求Config Service获取该namespace的最新配置。

# 客户端设计

![apollo-客户端](https://raw.githubusercontent.com/Leowuqunqun/img/master/imageapollo-%E5%AE%A2%E6%88%B7%E7%AB%AF.png)

1. 客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。（通过Http Long Polling实现）
2. 客户端还会定时从Apollo配置中心服务端拉取应用的最新配置。
   - 这是一个fallback机制，为了防止推送机制失效导致配置不更新
   - 客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回304 - Not Modified
   - 定时频率默认为每5分钟拉取一次，客户端也可以通过在运行时指定System Property: apollo.refreshInterval来覆盖，单位为分钟。
3. 客户端从Apollo配置中心服务端获取到应用的最新配置后，会保存在内存中
4. 客户端会把从服务端获取到的配置在本地文件系统缓存一份
   - 在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置
5. 应用程序可以从Apollo客户端获取最新的配置、订阅配置更新通知
# 使用注意事项

- PUBLIC空间的内容不宜过多，当关联的应用越来越多的时候，数据流量传输将会是瓶颈
- 更好的基于Namesapce隔离你的配置
- 基于接口去做测试和生产的自动化同步配置
# 参考
[https://www.apolloconfig.com/#/zh/README](https://www.apolloconfig.com/#/zh/README)

