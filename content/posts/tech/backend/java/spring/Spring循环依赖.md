---
title: "Spring循环依赖"
date: 2017-09-03
draft: false
layout: posts
tags: ["Java","Spring","循环依赖"]
---

# 什么是循环依赖问题？
循环依赖：说白是一个或多个对象实例之间存在直接或间接的依赖关系，这种依赖关系构成了构成一个**环形调用**。
第一种情况：自己依赖自己的直接依赖

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271019885.png)
第二种情况：两个对象之间的直接依赖

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271018210.png)
第三种情况：多个对象之间的间接依赖
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271019038.png)前面两种情况的直接循环依赖比较直观，非常好识别，但是第三种间接循环依赖的情况有时候因为业务代码调用层级很深，不容易识别出来。

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271008871.png)
# 

# 循环依赖的主要场景
## 单例的Setter注入
```java
@Service
public class TestService1 {

    @Autowired
    private TestService2 testService2;

    public void test1() {
    }
}

@Service
public class TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```
这是一个经典的循环依赖，但是它能正常运行，得益于spring的内部机制，让我们根本无法感知它有问题，因为spring默默帮我们解决了。
spring内部有三级缓存： 

- singletonObjects 一级缓存，用于保存实例化、注入、初始化完成的bean实例
- earlySingletonObjects 二级缓存，用于保存实例化完成的bean实例
- singletonFactories 三级缓存，用于保存bean创建工厂，以便于后面扩展有机会创建代理对象。

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271009543.png)
细心的朋友可能会发现在这种场景中第二级缓存作用不大。
那么问题来了，为什么要用第二级缓存呢？
试想一下，如果出现以下这种情况，我们要如何处理？

```java
@Service
public class TestService1 {

    @Autowired
    private TestService2 testService2;
    @Autowired
    private TestService3 testService3;

    public void test1() {
    }
}
@Service
public class TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
@Service
public class TestService3 {

    @Autowired
    private TestService1 testService1;

    public void test3() {
    }
}
```
TestService1依赖于TestService2和TestService3，而TestService2依赖于TestService1，同时TestService3也依赖于TestService1。
按照上图的流程可以把TestService1注入到TestService2，并且TestService1的实例是从第三级缓存中获取的。
假设不用第二级缓存，TestService1注入到TestService3的流程如图：
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271009697.png)

TestService1注入到TestService3又需要从第三级缓存中获取实例，而第三级缓存里保存的并非真正的实例对象，而是ObjectFactory对象。说白了，两次从三级缓存中获取都是ObjectFactory对象，而通过它创建的实例对象每次可能都不一样的。
这样不是有问题？
为了解决这个问题，spring引入的第二级缓存。上面图1其实TestService1对象的实例已经被添加到第二级缓存中了，而在TestService1注入到TestService3时，只用从第二级缓存中获取该对象即可。
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271009899.png)
还有个问题，第三级缓存中为什么要添加ObjectFactory对象，直接保存实例对象不行吗？
答：不行，因为假如你想对添加到三级缓存中的实例对象进行增强，直接用实例对象是行不通的。
针对这种场景spring是怎么做的呢？
答案就在AbstractAutowireCapableBeanFactory类doCreateBean方法的这段代码中：
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271009242.png)
它定义了一个匿名内部类，通过getEarlyBeanReference方法获取代理对象，其实底层是通过AbstractAutoProxyCreator类的getEarlyBeanReference生成代理对象。

## 多例的Setter注入
这种注入方法偶然会有，特别是在多线程的场景下，具体代码如下：
```java
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Service
public class TestService1 {

    @Autowired
    private TestService2 testService2;

    public void test1() {
    }
}

@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Service
public class TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```
很多人说这种情况spring[容器](https://cloud.tencent.com/product/tke?from=10680)启动会报错，其实是不对的，我非常负责任的告诉你程序能够正常启动。
为什么呢？
其实在AbstractApplicationContext类的refresh方法中告诉了我们答案，它会调用finishBeanFactoryInitialization方法，该方法的作用是为了spring容器启动的时候提前初始化一些bean。该方法的内部又调用了preInstantiateSingletons方法
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271010425.png)
标红的地方明显能够看出：非抽象、单例 并且非懒加载的类才能被提前初始bean。
而多例即SCOPE_PROTOTYPE类型的类，非单例，不会被提前初始化bean，所以程序能够正常启动。
如何让他提前初始化bean呢？
只需要再定义一个单例的类，在它里面注入TestService1

```java
@Service
public class TestService3 {

    @Autowired
    private TestService1 testService1;
}
```
重新启动程序，执行结果：
```java
Requested bean is currently in creation: Is there an unresolvable circular reference?
```
果然出现了循环依赖。
**注意：这种循环依赖问题是无法解决的，因为它没有用缓存，每次都会生成一个新对象。
**

## 构造器注入
## 单例的代理对象Setter注入
这种注入方式其实也比较常用，比如平时使用：@Async注解的场景，会通过AOP自动生成代理对象。
```java
@Service
public class TestService1 {

    @Autowired
    private TestService2 testService2;

    @Async
    public void test1() {
    }
}
@Service
public class TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```
```java
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'testService1': Bean with name 'testService1' has been injected into other beans [testService2] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.

```
为什么会循环依赖呢？

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271010794.png)
说白了，bean初始化完成之后，后面还有一步去检查：第二级缓存 和 原始对象 是否相等。由于它对前面流程来说无关紧要，所以前面的流程图中省略了，但是在这里是关键点，我们重点说说：
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271010308.png)
第二级缓存 和 原始对象不相等，所以抛出了循环依赖的异常。
如果这时候把TestService1改个名字，改成：TestService6，其他的都不变。

```java
@Service
publicclass TestService6 {

    @Autowired
    private TestService2 testService2;

    @Async
    public void test1() {
    }
}
```
再重新启动一下程序，神奇般的好了。 
what？ 这又是为什么？ 
这就要从spring的bean加载顺序说起了，默认情况下，spring是按照文件完整路径递归查找的，按路径+文件名排序，排在前面的先加载。所以TestService1比TestService2先加载，而改了文件名称之后，TestService2比TestService6先加载。
为什么TestService2比TestService6先加载就没问题呢？
答案在下面这张图中：
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271010844.png)
这种情况testService6中其实第二级缓存是空的，不需要跟原始对象判断，所以不会抛出循环依赖。 

# 思考题？出现循环依赖如何解决

