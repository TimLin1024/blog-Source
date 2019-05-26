---
title: Android 消息机制解析
date: 2017-04-28 11:50:11
tags: 
- 原理分析
- Android 进阶
categories: 
- 原理分析
- Android 进阶
---

# 概述

提起 Handler，相信大家都不会陌生，日常开发中不可避免地要使用到它。今天要讲的是消息机制。消息机制？咋听起来有点抽象。其实我们在使用 Handler 的时候，就使用了 Android 的消息机制。

从开发的角度来看，Handler 是 Android 消息机制的**上层接口**，这使得开发过程中只需要设计这方面的内容即可。但是作为一名合格的开发者，弄清它的实现原理是很有必要的。

<!-- more -->

俗话说得好，一图胜千言，我们先来看下 Android 消息机制简单示意图（图片参考自[这篇文章](http://blog.csdn.net/iispring/article/details/47180325)）。

![image](https://user-images.githubusercontent.com/16668676/44624139-24697380-a918-11e8-819a-82c87c7f7566.png)



我们把 Thread 比作是一个 发动机，MessageQueue 看作是一条流水线，Message 就像是流水线上的工人，Looper 是流水线下的滚筒，Handler 像是一个工人，它负责把 Message 这个产品送到流水线上，最后又负责把它取走。

这幅图中的各个组件的说明如下：

-   Looper ==》 滚轮
-   MessageQueue ==》 流水线
-   Message ==> 流水线上的产品
-   Handler ==》 扮演把『产品』送进流水线，完了以后又把『产品』取走的角色
-   Thread ==》 动力

另外还有一个重要概念 ——ThreadLocal 图中没有标明出来。下面对各个部分进行详细介绍。

# Android 的消息机制分析

## 从 Handler 出发

相信很多做 Android 开发的同学都写过与下面相似的代码。在子线程中做一些耗时操作，比如网络请求，操作完成之后，将返回的数据包装为 Message 对象然后调用 sendMessageXxx 方法，最后在 handleMessage 方法中对结果进行处理。

```java
Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case 1:
            	//handle 
                break;
            default:
                super.handleMessage(msg);
        }
    }
};

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

从 `Handler.sendMessage(message)`  到 `Handler.handlerMessage `方法经历了什么样的过程？

我们先看看 sendMessage 方法内部是怎么实现的。

```java
public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
  if (mAsynchronous) {
      msg.setAsynchronous(true);
  }
  return queue.enqueueMessage(msg, uptimeMillis);
}
    
```

整个调用流程是这样的 `sendMessage ==》 sendMessageDelayed ==》 sendMessageAtTime ==》 enqueueMessage ==》  MessageQueue.enqueueMessage`

我们可能还会调用 `Handler.post(Runnable) `方法到目标线程中执行 run 方法。post 方法会先调用 `getPostMessage `方法将 Runable 包装为 一个 Message 对象，（Runnable 就存储在 callback 中）。其他的 `postXxx(Runnable) `方法内部实现也是这样的流程，首先将 Runnable 包装为一个 Message 对象然后调用相应的 sendXxx 方法。

```java
public final boolean post(Runnable r){
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {//将 Runnable 包装为一个 Message 对象
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

从上述的调用流程可以看出 sendXxx  或者 postXxx 方法最终都会调用 MessageQueue 的 `enqueueMessage `方法，将 Message 追加到 MessageQueue 中。

### 小结

-   Handler 的消息的发送可以通过 post 的一系列方法以及 send 的一系列方法来实现。
    -   post 系列方法最终是通过 send 的一系列方法来实现的。

**Handler 的发送消息的过程仅仅是向消息队列插入了一条消息**。



### MessageQueue 对象是从哪里来的？

mQueue 是 Handler 的一个成员变量，它是在哪里初始化的呢？先看看 Handler 的构造方法

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
	//代码省略
    mLooper = Looper.myLooper();//获取当前线程的 Looper 
    if (mLooper == null) {
        throw new RuntimeException(//抛出异常，不能在没有 Looper 的线程创建 Handler 
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;//从本线程的 Looper 中获取 MessageQueue
    mCallback = callback;//回调
    mAsynchronous = async;//是否异步
}
```

我们看到 mQueue 是从 Looper 中取出的。在解说 Looper 之前，我们先看看前面提到的 MessageQueue 。



## MessageQueue 的工作原理

MessageQueue 主要包含两个操作：插入和读取（分别对应 enqueueMessage 和 next 方法）。

-   顾名思义，enqueueMessage 的作用是往队列中插入一条信息。
-   next() 的作用是从队列中取出一条信息并将其从消息队列中移除。

虽然 MessageQueue 名为消息队列，但是它的**内部实现并不是用队列**，而是通过一个**单链表的数据结构**来维护消息列表。

-   为什么选择使用单链表结构？ 因为 Message 是可以定时发送的，若使用普通的队列，当插入一个发送时间晚于队首 Message  发送时间的新 Message，那么就需要插队，实现起来不方便，而使用优先队列又显得比较复杂。因此就采用了单链表实现。

接下来我们重点看看 enqueueMessage 方法和 next 方法。

### enqueueMessage 方法：

```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        // 要进入队列的消息对象的目标 handler 不能为空
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        // 要进入队的消息不能处在使用状态
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {//入队需要先获取内置锁
        if (mQuitting) {
            // 已经调用过 Looper.quit / Looper.quitSafely 方法。 MessageQueue 中不能再追加 Message 对象
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();//回收消息
            return false;//返回 false 表示入队失败
        }
        // 标记消息为使用状态；设置消息发送的时间；是否需要唤醒
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //将消息插入到队首
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
          	//通过该循环找到合适的插入位置（以发送的时间作为排序的标准）
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
          	//插入队列的指定位置中
            msg.next = p; 
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);//当前处于阻塞状态，通过「写描述符」唤醒线程，从而使得 MessageQueue#next 方法中的 nativePollOnce 能够返回。（实际上运用了 Linux 的 epoll 机制）
        }
    }
    return true;
}
```

从代码中不难看出，enqueueMessaege 虽然有点长，但是逻辑还是比较清晰的。它先做了一些状态判断以及一些边界检测，完了以后再进行单链表的插入操作。

### next 方法

```java
  Message next() {
        final long ptr = mPtr;
        if (ptr == 0) {
            return null; 
        } 
		//代码省略
        for (;;) { 
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            } 
 			//处理 native 层事件（内部使用 Linux 的 epoll 机制），可能会阻塞
            nativePollOnce(ptr, nextPollTimeoutMillis);
 
            synchronized (this) {
                // Try to retrieve the next message.  Return if found. 
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {//如果消息的目标 Handler 为空
                    do { // 找出队列中下一个异步 Message 对象
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                } 
                if (msg != null) {
                    if (now < msg.when) {
                        // 计算下一条消息的执行时间，设置一个唤醒的延迟
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else { 
                        // 队首 Message 执行的时机到了，获取一条消息 
                        mBlocked = false; 
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else { 
                            mMessages = msg.next;
                        } 
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();//标记 Message 为正在使用状态
                        return msg;//返回消息
                    } 
                } else { 
                    // 队列中没有消息了
                    nextPollTimeoutMillis = -1;
                } 
 
                // 外部调用了 quit 方法。退出
                if (mQuitting) { 
                    dispose(); //处理底层消息队列。实际上移除了 native 层的消息队列
                    return null; 
                } 
 

                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                } 
                if (pendingIdleHandlerCount <= 0) {
                    // 没有可运行的闲置 handler。跳出本次循环再等待。
                    mBlocked = true; 
                    continue; 
                } 
				//代码省略
            }
    } 
```

next 方法中有一个死循环，其主要逻辑如下:

-   如果消息队列中没有消息，那么 next 会一直阻塞在那里。
    -   当队首的消息设置了延迟执行时，会造成短时间的阻塞。
-   当有新消息到来时，next 方法会返回这条信息并将其从单链表中移除。

当调用了 quit 方法之后，mQuitting 为 true ，next 方法会移除 native 层的消息队列并返回 null。

整个 next 方法的逻辑如下： 从消息队列中依次取出消息。如果这个消息到了执行时间，那么就将该消息返回给 Looper，并且将消息队列链表的指针后移。实际上消息队列维护着一个**分发屏障**(dispatch barrier)，当一个 Message 的时间戳低于这个值的时候，消息就会被分发给 Handler 进行处理。形象一点？看看下面这张图（图片来自 [Android Handler Internals](https://medium.com/@jagsaund/android-handler-internals-b5d49eba6977)，侵删）

![image](https://user-images.githubusercontent.com/16668676/32033334-a880e7ee-ba3e-11e7-8294-da929cc7bea0.png)

位于 dispatch barrier 左边的 Message 都被阻塞着，位于其右边的是即将分发的 Message。



我们是不是该说下 Looper 了，下下个就到它了，在此之前需要先看看 ThreadLocal 相关知识。

## ThreadLocal 简介

ThreadLocal  是一个**线程内部的数据存储类**，通过它可以在指定的线程中存储数据，数据存储后，只有在指定线程中才能获取到存储的数据，对于其他线程来说则无法获取到数据。

使用场景：

1. 当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，可以考虑采用 ThrradLocal。
    -   在 Android 中，Looper 类就是利用了 ThreadLocal 的特性，保证**每个线程只存在一个 Looper 对象**。
2. 复杂逻辑下的对象传递。
    -   比如监听器的传递。

从 ThreadLocal 的 set 和 get 方法 可以看出，它们所操作的对象都是**当前线程的 localValues 对象的 table 数组**，因此在不同的线程中访问同一个 ThreadLocal 的 get 和 set 方法，他们对 ThreadLocal 所做的读/写操作仅限于各自线程的内部。

关于 ThreadLocal 的具体介绍请见 [这篇文章](https://ivanljt.github.io/blog/2017/08/21/谈谈-ThreadLocal/#more)

理解 ThreadLocal 对后面理解 Looper 有很大的帮助，建议先细看 ThreadLocal  的内容再看后面的内容。



## Looper 的工作原理

Android 的官方文档中是这么介绍 Looper 的：

>   Looper 是一个用来为单个线程运行消息循环的类。**默认情况下线程是没有一个 Looper 跟他们相关联的**。如果一个线程需要 looper 的话，可以通过先调用 prepare() 方法初始化一个本线程的 Looper 实例。然后调用 loop 方法让它开始处理信息，一直到循环结束。

>   我们通常通过 Handler 类与 Looper 的打交道。

一个 线程最多只能有一个 Looper，一个 Looper 中有一个消息队列（前面 Hanlder 中的 MessageQueue 对象就是从 Looper 中取出的），并且持有它所在线程的引用。Looper 就像一个「死循环」（通过 quit 或者 quitSafely 方法可以退出），它会不断地从 MessageQueue 中查看是否有新消息。如果有，就调用 handler.dispatchMessage 方法进行处理；如果没有，就一直阻塞在那里。

### 创建 Looper

我们先看看 Looper 的构造方法：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);//创建一个消息队列
    mThread = Thread.currentThread(); //把当前线程的对象保存起来
}
```

Looper 构造方法中创建了一个消息队列，并且会保存当前线程对象。

#### 创建普通线程上的 Looper

Looper 的构造方法是私有的，那么要创建一个 Looper。？  
**调用 `Looper.prepare()`即可为当前线程创建一个 Looper 对象，接着通过 `Looper.loop()` 方法开启消息循环**。要注意的是：在子线程中，如果手动为其创建了 Looper，那么在所有的任务完成之后，需要调用 quit 方法来终止消息循环，否则这个子线程就会一直处于等待状态。

我们看下 prepare 方法实现

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
      throw new RuntimeException("Only one Looper may be created per thread");//一个线程最多只能有一个 Looper
    }
    sThreadLocal.set(new Looper(quitAllowed));//为当前线程创建一个 Looper 
}

public static @Nullable Looper myLooper() {//获取当前线程的 Looper 对象
    return sThreadLocal.get();
}
```

#### 创建主线程上的 Looper

有一个要注意的地方就是 Looper 的另一个创建方法 —— prepareMainLooper。

prepareMainLooper 在应用的入口方法（ActivityThread.main() ）中被调用，用来启动主线程的消息循环。

```java
public static void main(String[] args) {
    //代码省略
    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();//创建主线程的 Looper

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();//创建主线程的 Handler
    }
	//代码省略
    Looper.loop();//开启主线程消息循环
}
```



```java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

public static Looper getMainLooper() {//该方法使得在任何地方都可以获得主线程的消息循环
    synchronized (Looper.class) {
        return sMainLooper;
    }
}
```

#### 小结

在应用启动时会开启一个主线程（UI 线程），并且**开启消息循环**，应用不断地从该消息队列中取出、处理消息达到程序运行的结果。



### loop 方法

前面所讲都是 Looper 自身的一些特性，没有提到它是怎么跟其他部分交互的。Looper 与其他 MessageQueue 、Hanlder 的交互主要在 loop 方法中。下面我们来看看 loop 方法的源码。

```java
public static void loop() {
    final Looper me = myLooper();//获取本线程的 Looper
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");//调用 loop 方法之前必须先调用 Looper.prepare() 方法创建 Looper
    }
    final MessageQueue queue = me.mQueue;//获取所在线程的消息队列

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) { 
        Message msg = queue.next(); // 阻塞方法，如果没有获取到消息，就一直阻塞在这里
        if (msg == null) {  //只有当 msg == null 时才会退出循环
            return;
        }
        // 代码省略
        try {
            msg.target.dispatchMessage(msg);// 这里的 msg.target 是发送这条信息的 Handler 对象，这样 Handler 发送的消息最终又交给它的 dispatchMessage 方法来处理了。
        } finally {
            // 代码省略
        }

        // 代码省略

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        // 代码省略

        msg.recycleUnchecked();//调用 Message 的回收方法
    }
}
```



由源码可见 loop 方法是一个死循环，**唯一跳出循环的方式是 MessageQueue 的 next 方法返回了 null**。

当我们调用 Looper 的 quit 方法时，Looper 会调用 MessageQueue 的 quit 或 quitSafely 方法来通知消息队列退出，否则 loop 方法会一直循环下去。

另外，代码中可以看到，loop 调用了 MessageQueue 的 next 方法来获取新消息，而 next 是一个阻塞操作，当没有消息时，next 方法会一直阻塞在那里，这也导致了 loop 方法一直阻塞在那里。

`msg.target.dispatchMessage(msg);`// 这里的 msg.target 是发送这条信息的 Handler 对象，这样 **Handler 发送的消息最终又交给它的 dispatchMessage 方法来处理**了。

绕了这么一个大圈意义何在？  
通常我们都会在子线程中调用 Handler.sendMessageXxx 或者 Handler.postXxx 方法， 而**Handler 的 dispatchMessage 方法是在创建 Handler 的那个线程中执行的，这样就顺利地将代码切换到目标线程中去执行了**。





### 退出 Looper

Looper 也是可以退出的（这里的退出 Looper 主要是指跳出 loop 方法中的死循环）。通过调用 Looper 的 quit 方法或者 quitSafely 方法即可。那么这二者有什么区别？

-   调用 Looper 的 quit 方法可以**直接退出**Looper。
-   调用 Looper 的 quitSafely 方法只是设定了一个**退出标记**，然后把消息队列中**已有的消息处理完才退出** Looper。

我们来看看这两个方法的实现

```java
//Looper.quit
public void quit() {
    mQueue.quit(false);
}
//Looper.quitSafely
public void quitSafely() {
    mQueue.quit(true);
}
```

mQueue 的实际类型为 MessageQueue，Looper 的两个 quit 方法都是通过调用 MessageQueue 的 quit 方法来实现的。



```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");//主线程的  Looper 无法退出。主线程的 Looper 是通过 prepareMainLooper 方法创建的，创建时调用了 prepare（false），也就是令 mQuitAllowed = false
    }
    synchronized (this) {
        if (mQuitting) {//是否正在退出
            return;
        }
        mQuitting = true;

        if (safe) {
          	//安全退出,不会取消执行时机早于或等于当前时间的 Message
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();//移除 MessageQueue 中所有的消息
        }
        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

`MessageQueue.next()` 方法片段。当调用了 quit 方法之后会使得 mQuitting 为 true，从而导致  next 方法返回 null，一旦 next 方法返回 null， `Looper.loop` 就跳出了死循环。

```java
if (mQuitting) {
    dispose();//销毁 native 层的消息队列。
    return null;
}
```



 `MessageQueue.enqueueMessage ` 方法片段

```java
if (mQuitting) {
    IllegalStateException e = new IllegalStateException(
            msg.target + " sending message to a Handler on a dead thread");
    Log.w(TAG, e.getMessage(), e);
    msg.recycle();
    return false;
}
```

从这里实现可以看到，调用了 Looper.quit / quitSafely 方法之后，再通过 Handler 发送的消息无法添加到 MessageQueue 中，此时 Handler 的 send 方法会返回 false。



`android.os.MessageQueue#dispose`

```java
// 销毁底层的消息队列
// Must only be called on the looper thread or the finalizer.
//只能在不 looper 线程或者 finalizer 中调用
private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr);
        mPtr = 0;
    }
}
```



## 回到 Handler

 Looper 的 loop 方法中有这样一行代码 `msg.target.dispatchMessage(msg);`该方法调用的就是 Handler.dispatchMessage 方法。dispatchMessage 方法会根据情况对 Message 进行分发。

`Handler.class` 代码片段

```java
/**
 * Handle system messages here.
 */
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);//回调执行 Runnable 的 run 方法
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {//执行创建 Handler 时指定的 Callback#handleMessage 方法。
                return;
            }
        }
        handleMessage(msg);//执行 handleMessage(msg) 方法
    }
}

public void handleMessage(Message msg) {//空实现，需要由子类覆写
}

private static void handleCallback(Message message) {
    message.callback.run(); // callback 的实际类型为 Runnable，这行代码的作用就是回调 Runnable 的 run 方法
}

/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 */
public interface Callback {
    /**
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public boolean handleMessage(Message msg);
}

```

dispatchMessage 方法的逻辑：如果 **Message 自身**设置了 callback，那么事件会直接分派给 msg#callback#run  方法处理。否则，如果给 Handler 设置了 Callback 的话，会先将事件分派给 Callback#handleMessage 方法处理，如果返回值为 true，说明不需要再做进一步的处理，如果返回 false或者是没有给 Handler 设置callback 的 话，则会执行 Handler#handleMessage 方法。

Handler 的 Callback 有什么作用呢？

1. 提供了另一种使用 handler 处理消息的方式，在实例化 Handler 时可以使用Callback ，而不是自己实现一个 handler 子类。
2. 可以做一些消息过滤。



### Handler 小结

Handler 的工作主要包含**消息的发送**和**接收**过程（还有一个「分发过程」）。  

-   消息的发送可以通过 post 的一系列方法以及 send 的一系列方法来实现。
    -   post 系列方法最终是通过 send 的一系列方法来实现的。

查看源码不难发现，Handler 的发送消息的过程仅仅是向 MessageQueue 插入了一条 Message，MessageQueue 的 next 方法就会返回此 Message 给 Looper， Looper 在 loop 方法中对 Message 进行处理，最终由 Looper 交回给 Handler 处理（调用 Handler 的 dispatchMessage 方法）。



## 数量关系

一个线程最多只能有一个 Looper ，一个 MessageQueue，可以有多个 Handler。

MessageQueue 封装在 Looper 中。

## 问题

### 为什么主线程不会因为 Looper.loop()里的死循环卡死？ 

阻塞是有的，但是不会卡住
主要原因有 2 个：

#### 1.Linux 的 epoll 机制

当**没有消息**的时候会 **epoll.wait**，**等待句柄写的时候再唤醒**，这个时候其实是阻塞的。

#### 2.所有的 ui 操作都通过 handler 来发消息操作。

比如屏幕刷新 16ms 一个消息，你的各种点击事件，就会有**句柄写操作**，**唤醒上文的 wait 操作**，所以不会被卡死了。



#### 一个比较形象的比喻：

>   就像一辆车在圆形赛车道上跑，一边跑(一边执行任务)，比如开到市中心买瓶水，任务完成又回到刚刚离开的地方，继续各种执行买水的任务，直到没有任务了（msg 为空），行了，跑一圈回到起点。睡觉（epoll_wait），有任务叫醒你（唤醒 wait），你又开始跑一圈。边跑边接单。



想进一步了解的同学可以看下知乎上对该问题的[讨论](https://www.zhihu.com/question/34652589)

## 总结

Looper 对象封装了消息队列，Looper 对象被封装在 ThreadLocal 中，是线程私有的，不同线程之间的 Looper 无法共享。Handler 通过与 Looper 之间的绑定来实现与执行线程之间的绑定，handler 发送消息时会将 Message 对象追加到与线程相关的消息队列中，然后由 Looper 回调它的分发消息方法，根据情况处理消息。



最后我们看一张完整的流程图（图片参考自[Handler 异步通信机制全面解析](http://www.jianshu.com/p/9fe944ee02f7)），笔者修改了原图中的 Handler dispatchMessage  方法描述。

![](https://user-images.githubusercontent.com/16668676/29739304-0513811c-8a6d-11e7-8a78-510d3c98feb7.png)

## 参考资料与学习资源推荐

-   [理解 Java 中的 ThreadLocal ](http://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/index.html)
-   [谈谈 ThreadLocal](https://ivanljt.github.io/blog/2017/08/21/%E8%B0%88%E8%B0%88-ThreadLocal/#more)
-   [Android 中 Handler 的使用](http://blog.csdn.net/iispring/article/details/47115879)
-   [Handler 异步通信机制全面解析](http://www.jianshu.com/p/9fe944ee02f7)
-   [深入源码解析 Android 中的 Handler,Message,MessageQueue,Looper](http://blog.csdn.net/iispring/article/details/47180325)
-   [探索 Android 大杀器—— Handler](https://github.com/xitu/gold-miner/blob/master/TODO/android-handler-internals.md)
-   《Android 开发艺术探索》
-   《Android 源码设计模式解析与实战》

如果本文中有不正确的结论、说法或者表述不清晰的地方，恳请大家指出，共同探讨，共同进步，谢谢!