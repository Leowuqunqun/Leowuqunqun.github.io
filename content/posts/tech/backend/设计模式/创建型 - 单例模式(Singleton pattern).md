---
title: "创建型 - 单例模式(Singleton pattern)"
date: 2017-08-29
draft: false
layout: posts
tags: ["设计模式"]
---
> 前言内容阅读：Java内存模型

# 概念
保证一个类仅有一个实例，并提供一个访问它的全局访问点。 
## 原则

- 私有构造（阻止类被通过常规方法实例化）
- 以静态方法或者枚举返回实例（保证实例的唯一）
- 确保实例只有一个，尤其是多线程环境（保证在创建实例时的线程安全）
- 确保反序列化时不会重新构建对象（在有序列化反序列化的场景下防止单例被莫名其妙破坏，造成未考虑到的后果）
## 主要解决
单例模式主要解决的是，一个全局使用的类频繁的创建和销毁，从而提升提升整体的代码的性能。
## 使用场景

1. 数据库的连接池不会反复创建
2. spring中一个单例模式bean的生成和使用
3. 在我们平常的代码中需要设置全局的的一些属性保存
# 类图
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271116158.png)
# 6种实现
饿汉的意思是不管程序是否需要这个对象的实例，总是在类加载的时候就先创建好实例，理解起来就像不管一个人想不想吃东西都把吃的先买好，如同饿怕了一样。
懒汉的意思是如果一个对象使用频率不高，占用内存还特别大，明显就不合适用饿汉式了，这时就需要一种懒加载的思想，当程序需要这个实例的时候才去创建对象，就如同一个人懒的饿到不行了才去吃东西。
ps：乱七八糟的词汇真是让人不知道说啥好... 
## 饿汉模式 （线程安全）

```java
public class Singleton_03 {

    private static Singleton_03 instance = new Singleton_03();

    private Singleton_03() {
    }

    public static Singleton_03 getInstance() {
        return instance;
    }

}
```

- 在程序启动的时候直接运行加载，后续有外部需要使用的时候获取即可，空间换时间
- 导致的问题就像你下载个游戏软件，可能你游戏地图还没有打开呢，但是程序已经将这些地图全部实例化。到你手机上最明显体验就一开游戏内存满了，手机卡了，需要换了。
## 懒汉模式（线程安全）
```java
public class Singleton_02 {

    private static Singleton_02 instance;

    private Singleton_02() {
    }

    public static synchronized Singleton_02 getInstance(){
        if (null != instance) return instance;
        return new Singleton_02();
    }

}
```

- 虽然是安全的，但由于把锁加到方法上后，所有的访问都因需要锁占用导致资源的浪费。如果不是特殊情况下，不建议此种方式实现单例模式。
## 双重锁校验（线程安全）
```java
public class Singleton_05 {

    private volatile static Singleton_05 instance;

    private Singleton_05() {
    }

    public static Singleton_05 getInstance(){
       if(null != instance) return instance;
       synchronized (Singleton_05.class){
           if (null == instance){
               instance = new Singleton_05();
           }
       }
       return instance;
    }
}
```

- 双重锁的方式是方法级锁的优化，减少了部分获取实例的耗时。
- 同时这种方式也满足了懒加载。
- volatile关键字会强制的保证线程的可见性，而不加这个关键字，JVM也会尽力去保证可见性，但如果CPU一直处于繁忙状态就不确定了。
## 类的内部类（线程安全）
```java
public class Singleton_04 {
	
    private static class SingletonHolder {
        private static Singleton_04 instance = new Singleton_04();
    }

    private Singleton_04() {
    }

    public static Singleton_04 getInstance() {
        return SingletonHolder.instance;
    }
}
```

- 使用类的静态内部类实现的单例模式，既保证了线程安全有保证了懒加载，同时不会因为加锁的方式耗费性能。
- 这主要是因为JVM虚拟机可以保证多线程并发访问的正确性，也就是一个类的构造方法在多线程环境下可以被正确的加载。
- 此种方式也是非常推荐使用的一种单例模式
## CAS（线程安全）
```java
public class Singleton_06 {

    private static final AtomicReference<Singleton_06> INSTANCE = new AtomicReference<Singleton_06>();

    private static Singleton_06 instance;

    private Singleton_06() {
    }

    public static final Singleton_06 getInstance() {
        for (; ; ) {
            Singleton_06 instance = INSTANCE.get();
            if (null != instance) return instance;
            INSTANCE.compareAndSet(null, new Singleton_06());
            return INSTANCE.get();
        }
    }

    public static void main(String[] args) {
        System.out.println(Singleton_06.getInstance()); // org.itstack.demo.design.Singleton_06@2b193f2d
        System.out.println(Singleton_06.getInstance()); // org.itstack.demo.design.Singleton_06@2b193f2d
    }

}
```

- java并发库提供了很多原子类来支持并发访问的数据安全性；AtomicInteger、AtomicBoolean、AtomicLong、AtomicReference。
- AtomicReference 可以封装引用一个V实例，支持并发访问如上的单例方式就是使用了这样的一个特点。
- 使用CAS的好处就是不需要使用传统的加锁方式保证线程安全，而是依赖于CAS的忙等算法，依赖于底层硬件的实现，来保证线程安全。相对于其他锁的实现没有线程的切换和阻塞也就没有了额外的开销，并且可以支持较大的并发性。
- 当然CAS也有一个缺点就是忙等，如果一直没有获取到将会处于死循环中。
## 枚举单例（线程安全）
```java
public enum Singleton_07 {

    INSTANCE;
    public void test(){
        System.out.println("hi~");
    }

}
```

- 这种方式解决了最主要的；线程安全、自由串行化、单一实例。
- 这种写法在功能上与共有域方法相近，但是它更简洁，无偿地提供了串行化机制，绝对防止对此实例化，即使是在面对复杂的串行化或者反射攻击的时候。虽然这中方法还没有广泛采用，但是单元素的枚举类型已经成为实现Singleton的最佳方法。


