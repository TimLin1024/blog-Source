---
title: Android 源码中的享元模式——Message 复用原理
date: 2017-08-26 15:33:23
tags: 
- 原理分析
- Android 进阶
- 设计模式
categories: 
- 原理分析
- Android 进阶
- 设计模式
---

## 介绍

享元模式是对象池的一种实现，它的英文名为 Flyweight，代表轻量级的意思。

享元模式用来尽可能==减少内存使用量==，它适合用于可能存在大量对象的场景，来==缓存可共享的对象==（例如 Message、Java 中的字符串常量池）,从而实现对象共享、避免创建过多对象的效果，这样一来就可以提升性能，避免内存移除等。

享元模式中的部分状态是可以共享的，

-   可以共享的状态称为**内部状态**。内部状态不会随着环境变化
-   不可共享的状态则称之为**外部状态**，他们会随着环境的改变而改变。

享元模式会建立一个**对象容器**，在经典的享元模式中，该容器为一个 Map，它的键是享元对象的内部状态，它的值就是享元对象本身。

<!--more-->

## 享元模式的定义

享元模式是一种结构型设计模式，以共享的方式高效地支持大量的细粒度对象。

## 使用场景

1.  系统中存在**大量的相似对象**
2.  细粒度的对象都具备较接近的外部状态，而且内部状态与环境无关，也就是说对象没有特定身份。
3.  需要**缓冲池**的场景。

## UML 类图

![flyweight uml](https://user-images.githubusercontent.com/16668676/29702859-ac3f4f5c-89a5-11e7-92cd-2c26051b399f.png)



-   **Flyweight** ：享元对象抽象基类或者接口。
-   **ConcreteFlyweight**：具体享元对象。
-   **FlyweightFactory** ：享元工厂，负责创建享元对象和管理享元对象池。

## Android 源码中的享元模式

在使用 Handler 发送消息之前，我们一般都会使用如下代码调用 `mHandler.obtainMessage()` 方法获取一个 Message 对象。这其中究竟是怎么实现的呢？

```java
Handler mHandler = new Handler();

public void do() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            //do sth
            Message message = mHandler.obtainMessage();
            message.what = 1;
            message.obj = result;
            mHandler.sendMessage(message);
        }
    });
}
```

```java
//Handler.otainMessage()方法
public final Message obtainMessage(){
    return Message.obtain(this);
}
```

可以看到 Handler.obtainMessage() 实际上调用的是 Message 的 obtain 方法，我们顺着源码看下去。

先看看 Message 类部分源码

```java
// sometimes we store linked lists of these things
Message next;

private static final Object sPoolSync = new Object();//作为锁对象
private static Message sPool;//虽然名称为 sPool 但是实际上是一个指向消息队列队首的指针
private static int sPoolSize = 0;//

private static final int MAX_POOL_SIZE = 50;//「对象池」中的最大数量

public static Message obtain(Handler h) {
    Message m = obtain();//调用 obtain 方法获取 message 对象
    m.target = h;//指定 message 的目标对象
    return m;
}

//从消息对象池中取出一个 Message 对象，如果没有就创建一个
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // 清空 in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();//消息池中没有可复用的 Message 就创建一个新的 Message
}
```

至此，从对象池中获取对象的大致流程。无论是 Handler.obtainMessage(参数列表) 方法，还是 Message 的 obtain(参数列表) 方法，最终都会调用 Message.obtain() 方法。在 Message.obtain() 方法的实现中，会先从对象池中获取 Message 对象，如果获取不到，则创建一个新的 Message 对象，然后返回。该对象在后续的执行过程中会被回收到对象池，以便复用。

但是 Message 对象是如何被回收到「对象池」中的呢？    从 Message 类的部分代码中我们看到 sPool 的实际类型是一个 Message 对象，而不是一个容器。另外从 obtain 方法中我们不难看到链表的踪影。难道消息池是使用链表实现的吗？

在 AS 中打开 Message 类的结构图，可以看到其中有一个 recycle 方法，我们看看里面是怎么实现的。

```java
public void recycle() {
    if (isInUse()) {//判断消息是否还在使用
        if (gCheckRecycle) {//如果消息处在使用状态时被 gc 回收，就抛出异常
            throw new IllegalStateException("This message cannot be recycled because it " + "is still in use.");
        }
        return;//直接返回，取消回收操作
    }
    recycleUnchecked();//调用回收方法
}

/**
 * 回收一个可能还在使用的对象
 */
void recycleUnchecked() {
    // 只要该对象还在回收对象池中，就标记该对象为正在使用状态。
    // 清空其他状态
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
	//回收消息到消息池中
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

recycle 方法首先会判断 Message 对象是否处在使用状态。如果处在使用状态会直接返回（如果此时 GC 回收该对象会抛出异常），否则调用 recycleUnchecked 方法，具体的回收逻辑是在 recycleUnchecked  方法中实现的。首先会标记 Message 处于使用状态，然后清空对象中的其他状态。将消息存入回收池，主要是链表的操作。大致如下图所示。



![msg](https://user-images.githubusercontent.com/16668676/29739978-6c03d780-8a7e-11e7-8aad-3da3590c2ea1.png)



### 小结

Message 通过在内部构建一个链表来维护一个被会受到  Message 对象的对象池，当用户调用 obtain 方法时，会优先从池中获取。如果池中没有可以复用的对象就创建一个新的对象，该对象使用完之后，会被缓存到对象池中，当下次调用 obtain 方法时，他们就会被复用。

此处 Message 扮演了三个角色。既是 FlyWeight 抽象，又是 ConcreteFlyWeight 对象，同时还担任 FlyWeightFactory 角色，承担着管理对象池的职责。

想进一步了解 Android 消息机制的同学可参考[Android 消息机制解析](https://ivanljt.github.io/blog/2017/04/28/Android-%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E8%A7%A3%E6%9E%90/)。

## 总结

享元模式的优点：

-   大幅度降低了内存中对象的数量。从而降低了内存的占用，提高了程序的性能。

缺点：

-   使得系统更加复杂。为了使应用能够共享，需要将一些状态外部化，这使得程序的逻辑复杂化。
-   享元模式将状态外部化，而读取外部状态使得**运行时间稍微变长**

## 参考资料与学习资源推荐

-   [《JAVA 与模式》之享元模式](http://www.cnblogs.com/java-my-life/archive/2012/04/26/2468499.html)
-   《Android 源码设计模式解析与实战》
-   [Android 消息机制解析](https://ivanljt.github.io/blog/2017/04/28/Android-%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E8%A7%A3%E6%9E%90/)

若本文中有不正确的结论、说法，请大家指出，共同探讨，共同进步，谢谢!

