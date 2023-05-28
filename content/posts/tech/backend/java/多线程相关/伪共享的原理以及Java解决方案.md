---
title: "伪共享的原理以及Java解决方案"
date: 2016-04-25
draft: false
layout: posts
tags: ["Java","伪共享"]
cover: 
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262221226.png"
---



## 前言阅读
> 伪共享的诞生基于CPU多级缓存，建议阅读前言在看本文

[CPU多级缓存](../../../操作系统/操作系统之cpu多级缓存/)

## 伪共享的定义

伪共享不是单一语言问题，我在学习相关内容的时候，经常会发现某某语言伪共享，我们要清楚一件事情，伪共享的根本原因是操作系统层面，不同的语言都有针对伪共享的解决方案，本质上都是基于填充缓存行，以空间换时间
### CPU缓存行
缓存是由缓存行组成的，通常是 64 字节（常用处理器的缓存行是 64 字节的，比较旧的处理器缓存行是 32 字节），并且它有效地引用主内存中的一块地址。
一个 Java 的 long 类型是 8 字节，因此在一个缓存行中可以存 8 个 long 类型的变量。

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262221226.png)

在程序运行的过程中，缓存每次更新都从主内存中加载连续的 64 个字节。因此，如果访问一个 long 类型的数组时，当数组中的一个值被加载到缓存中时，另外 7 个元素也会被加载到缓存中。
但是，如果使用的数据结构中的项在内存中不是彼此相邻的，比如链表，那么将得不到免费缓存加载带来的好处。
不过，这种免费加载也有一个坏处。设想如果我们有个 long 类型的变量 a，它不是数组的一部分，而是一个单独的变量，并且还有另外一个 long 类型的变量 b 紧挨着它，那么当加载 a 的时候将免费加载 b。
看起来似乎没有什么毛病，但是如果一个 CPU 核心的线程在对 a 进行修改，另一个 CPU 核心的线程却在对 b 进行读取。
当前者修改 a 时，会把 a 和 b 同时加载到前者核心的缓存行中，更新完 a 后其它所有包含 a 的缓存行都将失效，因为其它缓存中的 a 不是最新值了。
而当后者读取 b 时，发现这个缓存行已经失效了，需要从主内存中重新加载。
**请记住，我们的缓存都是以缓存行作为一个单位来处理的，所以失效 a 的缓存的同时，也会把 b 失效，反之亦然。**

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262221422.png)
**这样就出现了问题，b 和 a 完全不相干，每次却要因为 a 的更新需要从主内存重新读取，它被缓存未命中给拖慢了，这就是传说中的伪共享。**
## 伪共享在Java中的解决方案
### Padding 方式
正确的方式应该将该对象属性分组，将一起变化的放在一组，与其他属性无关的属性放到一组，将不变的属性放到一组。这样当每次对象变化时，不会带动所有的属性重新加载缓存，提升了读取效率。在JDK1.8以前，我们一般是在属性间增加长整型变量来分隔每一组属性。被操作的每一组属性占的字节数加上前后填充属性所占的字节数，不小于一个cache line的字节数就可以达到要求：
```java
public class DataPadding{   
	long a1,a2,a3,a4,a5,a6,a7,a8;//防止与前一个对象产生伪共享 
	int value;  
	long modifyTime;
	long b1,b2,b3,b4,b5,b6,b7,b8;//防止不相关变量伪共享;  
	boolean flag;  
	long c1,c2,c3,c4,c5,c6,c7,c8;// 
	long createTime;   
	char key;   
	long d1,d2,d3,d4,d5,d6,d7,d8;//防止与下一个对象产生伪共享
}
```
**通过填充变量，使不相关的变量分开**
### Contended注解方式
JDK1.8 提供了注解 **@Contended** 用于解决伪共享问题，需要注意的是，如果业务代码需要使用该注解，要添加JVM参数
-XX:-RestrictContended。
默认填充宽度为128，若需要自定义填充宽度，则设置
-XX:ContendedPaddingWidth
具体的使用方式为：
```java
@sun.misc.Contended
public final static class Value {
  public volatile long value = 0L;
}
```
## 实际应用案例
### Disruptor
**RingBuffer类**
RingBuffer类（即上节中粉红色的圆环）的类关系图如下：
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262221304.png)
通过源码分析，RingBuffer的父类，RingBufferFields采用数组来实现存放线程间的共享数据。下图，第57行，entries数组。

前面分析过数组比链表、树更具有缓存友好性，此处不做细表。不使用LinkedBlockingQueue队列，是基于无锁机制的考虑。详细分析可参考，并发编程网的翻译。这里我们主要分析RingBuffer的继承关系中的填充，解决缓存伪共享问题。如下图： 
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262222981.png)
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262222401.png)
依据JVM对象继承关系中父类属性与子类属性，内存地址连续排列布局，RingBufferPad的protected long p1,p2,p3,p4,p5,p6,p7;作为缓存前置填充，RingBuffer中的protected long p1,p2,p3,p4,p5,p6,p7;作为缓存后置填充。这样任意线程访问RingBuffer时，RingBuffer放在父类RingBufferFields的属性，都是独占一行Cache line不会产生伪共享问题。如图，RingBuffer的操作字段在RingBufferFields中，使用rbf标识：
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262223083.png)
按照一行缓存64字节计算，前后填充56字节（7个long），中间大于等于8字节的内容都能独占一行Cache line，此处rbf是大于8字节的。
**Sequence类**
Sequence类用来跟踪RingBuffer和事件处理器的增长步数，支持多个并发操作包括CAS指令和写指令。同时使用了Padding方式来实现，如下为其类结构图及Padding的类。
Sequence里在volatile long value前后放置了7个long padding，来解决伪共享的问题。示意如图，此处Value等于8字节：
也许读者应该会认为这里的图示比上面RingBuffer的图示更好理解，这里的操作属性只有一个value，两个图相互结合就更能理解了。
**Sequencer的实现**
在RingBuffer构造函数里面存在一个Sequencer接口，用来遍历数据，在生产者和消费者之间传递数据。Sequencer有两个实现类，单生产者模式的实现SingleProducerSequencer与多生产者模式的实现MultiProducerSequencer。它们的类结构如图：
单生产者是在Cache line中使用padding方式实现，源码如下：
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262223003.png)
多生产者则是使用 sun.misc.Unsafe来实现的。如下图：
![](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305262223193.png)
### JDK1.8 ConcurrentHashMap的处理
java.util.concurrent.ConcurrentHashMap在这个如雷贯耳的Map中，有一个很基本的操作问题，在并发条件下进行++操作。因为++这个操作并不是原子的，而且在连续的Atomic中，很容易产生伪共享（false sharing）。所以在其内部有专门的数据结构来保存long型的数据:
```java
/* ---------------- Counter support -------------- */

    /**
     * A padded cell for distributing counts.  Adapted from LongAdder
     * and Striped64.  See their internal docs for explanation.
     */
    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
```
### JDK1.8 Thread 的处理
java.lang.Thread在java中，生成随机数是和线程有着关联。而且在很多情况下，多线程下产生随机数的操作是很常见的，JDK为了确保产生随机数的操作不会产生false sharing ,把产生随机数的三个相关值设为独占cache line。
```java
 // The following three initially uninitialized fields are exclusively
    // managed by class java.util.concurrent.ThreadLocalRandom. These
    // fields are used to build the high-performance PRNGs in the
    // concurrent code, and we can not risk accidental false sharing.
    // Hence, the fields are isolated with @Contended.

    /** The current seed for a ThreadLocalRandom */
    @sun.misc.Contended("tlr")
    long threadLocalRandomSeed;

    /** Probe hash value; nonzero if threadLocalRandomSeed initialized */
    @sun.misc.Contended("tlr")
    int threadLocalRandomProbe;

    /** Secondary seed isolated from public ThreadLocalRandom sequence */
    @sun.misc.Contended("tlr")
    int threadLocalRandomSecondarySeed;
```
