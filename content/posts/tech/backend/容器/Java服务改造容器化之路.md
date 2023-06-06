---
title: "Java服务改造容器化之路"
date: 2019-03-01
draft: true
layout: posts
tags: ["容器化实战"]
cover:
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/image1650192852582-29fe581f-30b0-4f3c-a67c-5467537d74c8.png"
---

> 背景：服务迁移到容器

# 容器内获取CPU核数的坑
早期的JDK版本中，Jdk1.8u102，当你使用Java的Runtime获取CPU数量时，在容器里面会返回容器所在宿主机的核数，而不是容器自身的：
```java
int cores = Runtime.getRuntime().availableProcessors();
```
解决方案：
这其实是JDK的一个问题，已经trace在[JDK-8140793](https://bugs.openjdk.java.net/browse/JDK-8140793)，原因是获取CPU核数是通过读取两个环境变量，其中

| **ENV** | **Description** |
| --- | --- |
| _SC_NPROCESSORS_CONF | number of processors configured |
| _SC_NPROCESSORS_ONLN | The number of processors currently online (available) |

其中_SC_NPROCESSORS_CONF 就是我们需要容器真实的CPU数量。
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image1650192852582-29fe581f-30b0-4f3c-a67c-5467537d74c8.png)

- 第一种办法是使用新版本的Jdku191以上的版本
- 第二种办法是使用自编译上面的源代码，通过LD_PRLOAD的方式将修改后的so文件加载进去Mock掉CPU的核数
## 参考链接
[https://aijishu.com/a/1060000000006797](https://aijishu.com/a/1060000000006797)



# Dubbo服务治理和Kubernetes冲突



