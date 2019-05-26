---
title: RxJava 订阅原理
date: 2018-06-24 18:52:01
tags: 
- 框架
categories:
- 框架
---



## 概述

RxJava 是什么呢？根据`RxJava`在`GitHub`上给出的描述 RxJava – Reactive Extensions for the JVM – a library for composing asynchronous and event-based programs using observable sequences for the Java 大致意思是：一个可以在 JVM上 使用的，是由异步的基于事件编写的通过使用可观察序列构成的一个库。 

关键词：`异步`，`基于事件`，`可观察序列`



本文主要讲述 RxJava 的订阅原理。

<!--more-->

## 示例：HelloRxJava

一般学一门新的编程语言都是先从打印的「hello word」开始，我们看看如何用 Rxjava 打印出「Hello RxJava」 并以之作为分析的实例。

```java
//创建 Observable
Observable<String> observable = Observable.create(new ObservableOnSubscribe<String>() {
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("Hello RxJava");
        emitter.onComplete();
    }
});
//订阅
observable.subscribe(new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
		System.out.println("onSubsribe call");
    }

    @Override
    public void onNext(String s) {
		System.out.println(s);//打印收到的文字，这里是 Hello RxJava
    }

    @Override
    public void onError(Throwable e) {
		e.printStackTrace();
    }

    @Override
    public void onComplete() {
		System.out.println("onComplete call");
    }
});
```



## Observerable 是如何创建的？

首先 new 了一个 ObservableOnSubscribe 对象，并实现其中的 subscribe 方法。该对象被传递给了 Observable#create 方法以创建 Observable（当然也有其他方法可以创建 Observable ，但是原理大同小异）。

`io.reactivex.Observable#create`

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");//不可为空
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

create 方法传递进来的 Observable 又传递给了 Observable 构造方法。

```java
final ObservableOnSubscribe<T> source;//保存成员变量
public ObservableCreate(ObservableOnSubscribe<T> source) {
    this.source = source;
}
```

io.reactivex.plugins.RxJavaPlugins#onAssembly(io.reactivex.Observable<T>)

```java
/**
 * Calls the associated hook function.
 * @param <T> the value type
 * @param source the hook's input value
 * @return the value returned by the hook
 */
@SuppressWarnings({ "rawtypes", "unchecked" })
@NonNull
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

io.reactivex.plugins.RxJavaPlugins#onAssembly 方法中只是调用了相关的 hook 函数（如果有的话），然后返回原对象。



创建的结果：ObservableOnSubsribe 外面包装了一层。如下图所示：

![image-20180624202952875](https://ws2.sinaimg.cn/large/006tNc79gy1fsmjkz2m4fj307n04qglj.jpg)



## 订阅过程

实例代码中在订阅之前我们先创建了一个 `observer` 对象。

然后调用 `Observable#subscribe` 方法，将 `observer` 作为参数传递给该方法。

点开看看 `Observable#subscribe` 方法的实现。

```java
@Override
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");//非空检查
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");//非空检查

        subscribeActual(observer);//调用真正的订阅方法。
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);//

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```

上面 try 块中的第一行代码调用了 `RxJavaPlugins#onSubscribe` 。点开看看它具体做了啥？

```java
@SuppressWarnings({ "rawtypes", "unchecked" })
@NonNull
public static <T> Observer<? super T> onSubscribe(@NonNull Observable<T> source, @NonNull Observer<? super T> observer) {
    BiFunction<? super Observable, ? super Observer, ? extends Observer> f = onObservableSubscribe;
    if (f != null) {
        return apply(f, source, observer);//调用相应的 hook 函数(如果有的话)
    }
    return observer;
}
```



我们订阅的实际对象是 `ObserverableCreate`，因此点进去看看其中的 `subscribeActual` 方法实现：

`ObservableCreate#subscribeActual`

```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);//以 observer 作为参数构造一个 CreateEmitter
    observer.onSubscribe(parent);//回调 Observer#onSubsribe 方法

    try {
        source.subscribe(parent);//调用源订阅方法，也就是我们自己实现的在其中发送数据的方法
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```



这里的` source#subcribe` 也就是：

```java
public void subscribe(ObservableEmitter<String> emitter) throws Exception {
    emitter.onNext("hello rxJava");//调用 发送数据
    emitter.onComplete();
}
```



订阅是从下游传递到上游。传递到源头之后，会触发调用 `Emitter#onXxx` 方法，将数据从上游发送到下游。

### emitter 是如何将数据发射给 observer 的呢？

我们先看看 **emitter 是什么**？

上面的 emitter 的实际类型是 CreateEmitter

具体实现如下：

```java
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {


    private static final long serialVersionUID = -3434801548987643227L;

    final Observer<? super T> observer;

    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }

    @Override
    public void onNext(T t) {
        if (t == null) {//判空
            onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
            return;
        }
        if (!isDisposed()) {//如果当前状态不是 dispose
            observer.onNext(t);//调用 observer#onNext
        }
    }

    @Override
    public void onError(Throwable t) {
        if (!tryOnError(t)) {
            RxJavaPlugins.onError(t);
        }
    }

    @Override
    public boolean tryOnError(Throwable t) {
        if (t == null) {
            t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
        }
        if (!isDisposed()) {
            try {
                observer.onError(t);
            } finally {
                dispose();
            }
            return true;
        }
        return false;
    }

    @Override
    public void onComplete() {
        if (!isDisposed()) {//不是出于 disposed 状态
            try {
                observer.onComplete();//调用
            } finally {
                dispose();//onComplete 只能调用一次，调用完成之后，状态变为 dispose
            }
        }
        //代码省略
}
```

`CreateEmitter` 是 `ObservableCreate`的一个内部类，继承自`AtomicReference<Disposable>`, 实现了两个接口 `ObservableEmitter<T>` 和 `Disposable ` 。 

可以看到 `CreateEmitter` 对 `observer` 进行了包装（observer 依赖通过构造函数参数注入）。它在调用 observer 的相应方法的前后对状态进行判断和更新。



### `CreateEmitter` 又是在什么时候创建的呢？

在订阅过程中调用到 `ObservableCreate#subscribeActual` ，该方法会利用 observer 构造一个 `CreateEmmiter`， 然后把它作为参数去调用 ` source#subcribe` 方法。

`source` 也就是我们创建的 `ObservableOnSubscribe` 匿名内部类。`CreateEmmiter` 就是通过这样的方式作为参数传递给了我们自己实现的 `subscribe` 方法。

## 小结

订阅是从下游传递到上游。传递到源头之后，会触发调用 `Emitter#onXxx` 方法，将数据从上游发送到下游。

上游对数据流的控制是通过 `CreateEmitter` 实现的。



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！