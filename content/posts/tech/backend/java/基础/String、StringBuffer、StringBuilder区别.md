---
title: "String、StringBuffer、StringBuilder区别"
date: 2014-04-28
draft: false
layout: posts
tags: ["Java","StringBuffer","StringBuilder"]
cover: 
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271018261.png"
---

### 区别：
String是不可变的对象, 因此在每次对**String 类型**进行改变的时候，都会生成一个新的 String 对象，然后将指针指向新的 String 对象，所以经常改变内容的字符串最好不要用 String ，因为每次生成对象都会对系统性能产生影响，特别当内存中无引用对象多了以后， JVM 的 GC 就会开始工作，性能就会降低。
使用 **StringBuffer 类**时，每次都会对 StringBuffer 对象本身进行操作，而**不是**生成新的对象并改变对象引用。所以多数情况下推荐使用 StringBuffer ，特别是**字符串对象经常改变的情况**下。
### 为什么StringBuffer是线程安全，而StringBuilder不是？
StringBuffer的方法实现线程安全是通过Synchronized字段来实现的，[Synchronized原理](https://zhuanlan.zhihu.com/p/486514106)看这里
StringBuilder的append方法
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271018261.png)
编辑切换为居中
添加图片注释，不超过 140 字（可选）
StringBuilder的append方法调用了父类的append方法
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271018559.png)
编辑切换为居中
添加图片注释，不超过 140 字（可选)
我们直接看第七行代码，count += len; 不是一个原子操作，实际执行流程为 

- 首先加载count的值到寄存器
- 在寄存器中执行 +1操作
- 将结果写入内存

假设我们count的值是10，len的值为1，两个线程同时执行到了第七行，拿到的值都是10，执行完加法运算后将结果赋值给count，所以两个线程最终得到的结果都是11，而不是12，这就是最终结果小于我们预期结果的原因。
### 扩容机制

StringBuffer和StringBuilder都是继承自AbstractStringBuilder，所以他们的扩容机制是相同的
无参构造初始容量为：16，有参构造初始容量为：字符串参数的长度+16

- 一次追加长度超过当前(初始)容量，则会按照 当前(初始)容量*2+2 扩容一次
- 一次追加长度不仅超过初始容量，而且按照 当前容量*2+2 扩容一次也不够，其容量会直接扩容到与所添加的字符串长度相等的长度。之后再追加，还会按照 当前容量*2+2进行扩容
### 总结
1.如果要操作少量的数据用 = String
2.单线程操作字符串缓冲区 下操作大量数据 = StringBuilder
3.多线程操作字符串缓冲区 下操作大量数据 = StringBuffer
4.不要使用String类的"+"来进行**频繁的拼接**，因为那样的性能极差的，应该使用StringBuffer或StringBuilder类，这在Java的优化上是一条比较重要的原则
5.为了获得更好的性能，在构造 StirngBuffer 或 StirngBuilder 时应尽可能**指定它们的容量**。当然，如果你操作的字符串长度（length）不超过 16 个字符就不用了，当不指定容量（capacity）时默认构造一个容量为16的对象。不指定容量会显著降低性能。
6.在现实的模块化编程中，负责某一模块的程序员不一定能清晰地判断该模块是否会放入多线程的环境中运行，因此：除非确定系统的瓶颈是在 StringBuffer 上，并且确定你的模块不会运行在多线程模式下，才可以采用StringBuilder；否则还是用StringBuffer。
