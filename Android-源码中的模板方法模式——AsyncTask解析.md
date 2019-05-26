---
title: Android 源码中的模板方法模式——AsyncTask解析
date: 2017-08-24 11:47:11
tags: 
- 原理分析
- Android 进阶
- 设计模式
categories: 
- 原理分析
- Android 进阶
- 设计模式
---

## 前言

假设我们知道一个算法所需的关键步骤，并确定了这些步骤的执行顺序，但是，**某些步骤的具体实现是未知的**，或者说某些步骤的实现是会**随着环境的变化而改变的**。
就好像执行程序的流程：
1. 检查代码的正确性
2. 链接相关代码
3. 编译相关代码
4. 执行程序

对于不同的语言，上述 4 个步骤都是不一样的，但是它们的执行流程是固定的，这类问题的解决方案就是我们介绍的模板方法模式。

<!--more-->

## 模板方法模式的定义

定义一个操作中的**算法的框架**，而将一些步骤延迟到子类中，使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

## 模板方法模式的使用场景

1. 多个子类有公有的方法，并且逻辑基本相同时。
2. 重要、复杂的算法，可以把核心算法设计为模板方法，周边的相关细节功能则由各个子类实现。
3. 重构时，把相同的代码抽取到父类中，然后通过**钩子方法**约束其行为。


### 注：何谓钩子方法？

基本方法又可以分为三种：抽象方法（Abstract Method）、具体方法（Concrete Method）和钩子方法（Hook Method）。

这是《java与模式》书里的一种说法，三种方法也是在书中的模板方法模式中提及到的。

先说这个三个方法的基本定义：
-  抽象方法：由抽象类声明，由具体子类实现。在java语言里一个抽象方法以abstract关键字标示出来。
-  具体方法：由抽象类声明并实现，而子类并不实现或覆盖。其实就是**一般的方法**，但是不需要子类来实现。
-  钩子方法：由抽象类**声明并实现**，而子类也会加以扩展。通常抽象类给出的是一个空的钩子方法，也就是方法体为空的方法（也可以根据需要实现部分逻辑）。其实它和具体方法在代码上没有区别，不过是意识上的一种区别。


详见[抽象方法 具体方法 钩子方法](https://www.the5fire.com/%E6%8A%BD%E8%B1%A1%E6%96%B9%E6%B3%95-%E5%85%B7%E4%BD%93%E6%96%B9%E6%B3%95-%E9%92%A9%E5%AD%90%E6%96%B9%E6%B3%95.html)

## 模板方法模式的 UML 类图

![template method pattern](https://user-images.githubusercontent.com/16668676/29603702-be32185a-8817-11e7-95f3-9cf7ee08c644.png)

- AbsTemplate：抽象类，定义一套算法框架
- ConcreteImplA：具体实现类 A
- ConcreteImplB：具体实现类 B


## 模板方法模式的简单示例 
实际上是封装一个固定流程，就像是一套执行模板一样，第一步该做什么，第二步该做什么都已经在抽象类中定义好。而子类可以有不同的算法实现，在框架不被修改的情况下实现某些步骤的算法替换。

## Android 源码中的模板方法模式 
### AsyncTask

使用过 AsyncTask 的同学都知道，我们调用 execute 之后，（如果没有调用 cancel 方法的话）以下三个方法会依次执行：
- onPreExecute
- doInBackground 
- onPostExecute

为什么能让它们依次执行呢？其内部是怎么实现的？我们看看源码，一探究竟。

首先看看异步任务的入口方法 execute。
```java
 @MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}


@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
以上两个构造方法中主要做了如下几件事：
- 状态判断
- 判断之后执行 `onPreExecute();`
- 使用线程池执行 mFuture
    - 什么样的线程池？
        - 默认为 SerialExecutor 即单线程的线程池

```java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);//设置进程优先级
                //noinspection unchecked
                result = doInBackground(mParams);//调用 doInBackground 方法
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);//调用 postResult 方法
            }
            return result;
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());//任务完成
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```
```java
private final WorkerRunnable<Params, Result> mWorker;
private final FutureTask<Result> mFuture;
```

- mWorker 类型为 `WorkerRunnable<Params, Result>` ， WorkerRunnable 实现了 Callable
- mFuture 类型为 `FutureTask<Result>`

简而言之，这个 mFuture 包装了这个 mWorker 对象，而 mFuture 是在线程池中执行的，会调用 mFuture 的 run 方法，该 run 方法中调用了 mWorker 的 call 方法，mWorker 的 call 方法又调用了 doInBackground 方法，所以 doInBackground 是在工作线程执行的。

```java
private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}
```
`doInBackground` 执行完成后会通过 `postResult(result)` 方法将结果传递给主线程。
-  `postResult(result)` 可能通过 call 方法的 finally 块直接调用或者通过 FutureTask 中的 done 方法里面的 `postResultIfNotInvoked(get());` 来间接调用。

接下来我们看看 `postResult(result)` 方法
```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```
```java
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // 调用 AsyncTask 的 finish 方法
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
`postResult(result)`  方法就是通过发送一条消息（msg.what == MESSAGE_POST_RESULT）给 sHandler，sHandler 为 InternalHanlder。当 InternalHanlder 接收到 MESSAGE_POST_RESULT 时，就会调用 `result.mTask.finish(result.mData[0])` 方法，result 的类型为 AsyncTaskResult


```java
private static class AsyncTaskResult<Data> {
    final AsyncTask mTask;
    final Data[] mData;

    AsyncTaskResult(AsyncTask task, Data... data) {
        mTask = task;
        mData = data;
    }
}
```
从 AsyncTaskResult 的具体实现中吗，我们知道 mTask 就是 AsyncTask，finish 方法中又调用了 `onPostExecute` ，此时整个执行流程就完成了。

#### 小结

execute 方法内部封装了 onPreExecute、doInBackGround、onPostExecute 这个逻辑流程。
通过这种方式，用户可以根据自己的需求再覆写这几个方法，使得用户可以很方便地使用异步任务来完成耗时的操作及更新 UI。实际上就是通过线程池来执行耗时的任务，得到结果之后，通过 Handler 将结果传递给 UI 线程执行。

### Activity 的生命周期函数

除了 AsyncTask 以外，Android 源码中还有不少地方有模板方法的身影，比如说 Activity 的生命周期方法—— onCreate 、onStart、onResume 等，都是按照顺序调用的，我们会在对应的方法中执行合适的操作。

其内部实现涉及到进程间通信，限于篇幅，本文不作深入介绍。有兴趣的同学可以看看 ActivityThread 的 main 方法，以之作为入口，对生命周期方法的调用时机做进一步研究。

## 模板方法总结

简单概括模板方法模式就是流程封装。把某一个固定的流程封装到一个固定的 final 方法中。并且让子类能够定制这个过程中的某些甚至所有步骤，这就要求父类提取共用的代码，提升代码的复用率，同时也带来了更高的可扩展性。

- 优点：
    - 封装不变的部分，扩展可变的部分
    - 提取公共部分代码，便于维护。
- 缺点：
    - 提高了代码阅读的难度，会让用户觉得难以理解




## 参考资料与学习资源推荐

- 《Android 源码设计模式解析与实战》
- [抽象方法 具体方法 钩子方法](https://www.the5fire.com/%E6%8A%BD%E8%B1%A1%E6%96%B9%E6%B3%95-%E5%85%B7%E4%BD%93%E6%96%B9%E6%B3%95-%E9%92%A9%E5%AD%90%E6%96%B9%E6%B3%95.html)




由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！