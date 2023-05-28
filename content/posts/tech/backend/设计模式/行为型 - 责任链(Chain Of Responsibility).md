---
title: "行为型 - 责任链(Chain Of Responsibility)"
date: 2017-09-20
draft: false
layout: posts
tags: ["设计模式"]
cover: 
    image: "https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271117325.png"
---
## 概念
使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链发送该请求，直到有一个对象处理它为止。
## 应用场景

- SprintBoot 里的拦截器、过滤器链
- Okhttp 里的请求拦截器
- 在请求处理者不明确的情况下向多个对象中的一个提交请求
- 如果有多个对象可以处理同一个请求，但是具体由哪个对象处理是由运行时动态决定的，这种对象就可以使用职责链模式
##  类图
职责链模式主要包含以下角色。

1. 抽象处理者（Handler）角色：定义一个处理请求的接口，包含抽象处理方法和一个后继连接。
2. 具体处理者（Concrete Handler）角色：实现抽象处理者的处理方法，判断能否处理本次请求，如果可以处理请求则处理，否则将该请求转给它的后继者。
3. 客户类（Client）角色：创建处理链，并向链头的具体处理者对象提交请求，它不关心处理细节和请求的传递过程。

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271117325.png)

![img](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271117494.png)

## 优点

- 只需要将请求发送到责任链上即可，无须关心请求的处理细节和请求的传递，所以责任链将请求的发送者和请求的处理者降低了耦合度
- 通过改变链内的调动它们的次序，允许动态地新增或者删除处理类，比较方便维护
- 增强了系统的可扩展性，可以根据需要增加新的请求处理类，满足开闭原则
- 每个类只需要处理自己该处理的工作，明确各类的责任范围，满足单一职责原则

## 缺点

- 处理都分散到了单独的职责对象中，每个对象功能单一，要把整个流程处理完，需要很多的职责对象，会产生大量的细粒度职责对象
- 不能保证请求一定被接收，如果链路比较长，系统性能将受到一定影响，而且在进行代码调度时不太方便
## 实现
```java
public abstract class Handler {
	protected Handler successor;
	
	public Handler(Handler successor) {
		this.successor = successor;
	}
	
	protected abstract void handleRequest(Request request);
}
  
```
```java
public class ConcreteHandler1 extends Handler {
    public ConcreteHandler1(Handler successor) {
        super(successor);
    }

    @Override
    protected void handleRequest(Request request) {
        if (request.getType() == RequestType.type1) {
            System.out.println(request.getName() + " is handle by ConcreteHandler1");
            return;
        }
        if (successor != null) {
            successor.handleRequest(request);
        }
    }
}

```
```java
public class ConcreteHandler2 extends Handler{
    public ConcreteHandler2(Handler successor) {
        super(successor);
    }

    @Override
    protected void handleRequest(Request request) {
        if (request.getType() == RequestType.type2) {
            System.out.println(request.getName() + " is handle by ConcreteHandler2");
            return;
        }
        if (successor != null) {
            successor.handleRequest(request);
        }
    }
}

```
```java
public class Request {
    private RequestType type;
    private String name;

    public Request(RequestType type, String name) {
        this.type = type;
        this.name = name;
    }

    public RequestType getType() {
        return type;
    }

    public String getName() {
        return name;
    }
}

```
```java
public enum RequestType {
    type1, type2
}
```
```java
public class Client {
    public static void main(String[] args) {
        Handler handler1 = new ConcreteHandler1(null);
        Handler handler2 = new ConcreteHandler2(handler1);
        Request request1 = new Request(RequestType.type1, "request1");
        handler2.handleRequest(request1);
        Request request2 = new Request(RequestType.type2, "request2");
        handler2.handleRequest(request2);
    }
}
```
```java
request1 is handle by ConcreteHandler1
request2 is handle by ConcreteHandler2
```
