---
title: 拆轮子系列——LeakCanary工作原理
date: 2017-12-15 13:28:02
tags: 
- 源码分析
- 框架原理
categories:
- 源码分析
- 框架原理
---



## 一、概述

LeakCanary 是一个内存泄漏的自动化监测工具。

### 1.1什么是内存泄漏？

内存泄漏指的是应该被释放的内存的没有被释放。原因？长生命周期的对象持有短生命周期对象的引用。

### 1.2哪些对象应该被回收？

首先要找出哪些对象是**应该被回收**的？Android 中使用最频繁的是 Activity 和 Fragment 了。他们都有 onDestory 方法，当它们的 **onDestroy 执行的时候，就可以把它们列为 「应该被回收的对象**」，可在 onDestory  方法中建立**检测点**。

<!--more-->

注：

1.  `Application#dispatchActivityDestroyed` 是在 `Activity#onDestroy` 方法执行后回调的。

### 1.3如何知道对象是否被回收了？

手动 GC +ReferenceQueue + WeakReference

>   WeakReference 创建时，传入一个 ReferenceQueue 对象。当被 WeakReference 引用的对象的生命周期结束，一旦被 GC 检查到，GC 将会把该对象添加到 ReferenceQueue 中，待 ReferenceQueue 处理。当 GC 过后对象一直不被加入 ReferenceQueue，说明它可能存在内存泄漏。

这里其实有一个默认的前提就是，当一个对象存在强引用的时候，这个对象是不会被回收的，所以 GC 前后，可达性并不会发生变化，也就不会被加入到 referenceQueue 中。而如果一个对象只含有弱引用的时候，GC 前后可达性会发生改变——GC 之前弱可达，GC 之后变不可达。



什么样的对象会进入ReferenceQueue？

> 创建 Reference 的时候指定了 ReferenceQueue，并且对象的可达性发生了变化。

具体加入的时间是在 GC前还是 GC后 并不是很重要，重要的是，**一旦被添加到 ReferenceQueue 中，对应的对象一定会被回收** (及时被回收了自然也就没有内存泄漏问题了，所以 LeakCanary 把加入 ReferenceQueue 作为内存泄漏检测的初步判断标准)



假设没有发生内存泄漏，那么这个时候，Activity 仅被我们创建的 KeyedWeakReference 弱引用了。我们第一次手动 GC 的时候，它就会进入引用队列。这个时候可以将它从 retainkeys 中移除。

对象一旦只存在弱引用，会 ReferenceHandler 线程监听到，该线程会将该对象的引用加入 ReferenceQueue 中，这发生在 finalization 或者 gc 之前。



LeakCanary 判断是否发生内存的泄漏的标准：对象的引用是否在`Set<String> retainedKeys ` 中。

- 创建一个跟踪对象，将它的 key 存储在一个 retainedKeys 中。当对象出现在引用队列里面的时候，将它从 set 中移除，如果一个对象的引用不在retainKeys 中，说明没有发生内存泄漏。如果在，则说明可能发生了内存泄漏。



### 1.4未被回收==内存泄漏？

#### 没被回收的原因：被持有强引用了吗？

未被回收的对象，是否被其他对象引用？找出**其最短引用链**。`VMDebug` + `HAHA` 完成需求。

VMDebug、HAHA。

>   VM 会有堆内各个对象的引用情况，并能以`hprof`文件导出。HAHA 是一个由 square 开源的 Android 堆分析库，分析 `hprof` 文件生成`Snapshot`对象。`Snapshot`用以查询对象的最短引用链。



找到最短引用链后，定位问题，排查代码将会事半功倍。

下面是一个总体流程图：（流程图参考自[这篇文章](http://www.jianshu.com/p/3f1a1cc1e964)）

![](http://upload-images.jianshu.io/upload_images/737949-cff06b0079bf09c4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2)



如果读者还没看过 LeakCanary 的源码，建议照着上面的调用流程图过一遍，这样效果会好很多。why？[为什么源码分析味同嚼蜡？浅析技术写作中的思维误区](https://juejin.im/post/5a137248f265da43333e0418)

## 二、具体流程

前面讲了大体流程，接下来我们看看内部的实现。



通常会在 Applcation#onCreate 中对 LeakCanary 进行初始化,也就是调用下面的 setupLeakCanary 方法。

```java
protected RefWatcher setupLeakCanary() {
    if (LeakCanary.isInAnalyzerProcess(this)) {
        return RefWatcher.DISABLED;
    }
    return LeakCanary.install(this);
}
```



### 2.1LeakCanary#install 调用流程

```java
--> ① LeakCanary#install
	--> ② refWatcher
		--> ③ AndroidRefWatcherBuilder#listenerServiceClass	//监听服务类
			--> ④ AndroidExcludedRefs#createAppDefaults//创建默认的忽略列表
				--> ⑤ ExcludedRefs.Builder#build
			--> ⑥ RefWatcherBuilder#excludedRefs//排除一些引用
			--> ⑦ AndroidRefWatcherBuilder#buildAndInstall//构造一个 RefWatcher 
				--> ⑧ RefWatcherBuilder#build
				--> ⑨ ActivityRefWatcher#install
					--> ⑩ new ActivityRefWatcher
						--> ⑪ ActivityRefWatcher#watchActivities
						--> ⑫ stopWatchingActivities();//防止装载两次，
							--> ⑬ Application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks);//解注册
						--> ⑭    
              Application#registerActivityLifecycleCallbacks(lifecycleCallbacks);//注册生命周期回调
							--> ⑮1mActivityLifecycleCallbacks.add(callback);//添加到监听者列表中
```

注：监听回调进行仅仅实现了 

```java
@Override public void onActivityDestroyed(Activity activity) {
  ActivityRefWatcher.this.onActivityDestroyed(activity);
}
```

从上面可以看出在默认情况下只监听 Activity 的 `onDestory` 方法，也就是说只是检测 Activity 是否存在内存泄漏。

#### 如果要检测 Fragment 的内存泄漏，应该如何实现？

因为 Android 中只提供了 Activity 的生命周期方法的回调，而没有提供 Fragment 生命周期回调的监听。 如果要对 Fragment 的内存泄漏进行检测，那么需要自己在 `Fragment#onDestroy` 方法中手动创建一个 RefWatcher ，然后调用 `refWatcher#watch(this)`，最好是定义一个 Fragment 基类，在其中的 onDestroy 方法中定义相应的操作。

```java
public abstract class BaseFragment extends Fragment {

  @Override public void onDestroy() {
    super.onDestroy();
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(this);
  }
}
```







### 2.2当监听事件发生时，切换到后台执行

```java
--> ① ActivityLifecycleCallbacks#onActivityDestroyed
	-->② ActivityRefWatcher#onActivityDestroyed
		-->③ RefWatcher#watch(Object)
			-->④ RefWatcher#watch(Object, String)    内部有一个 ReferenceQueue<Object> queue;
				-->⑤ new KeyedWeakReference(watchedReference, key, referenceName, queue)//创建 KeyedWeakReference
  					//两个 Handler,mainHandler 、backgroundHandler
  				-->⑥ RefWatcher#ensureGoneAsync
  					-->⑦ WatchExecutor#execute 
 						if 当前线程为主线程
 							==> ⑧AndroidWatchExecutor#waitForIdle// 等待 MainLooper 空闲时发送
 							
								--> ⑨MessageQueue#addIdleHandler//当 Looper 即将闲置时发送
									--> ⑩AndroidWatchExecutor#postToBackgroundWithDelay
										--> ⑪backgroundHandler#postDelayed//工作线程 Handler 延迟发送
											--> ⑫Retryable
											--> if (result == RETRY) 
         										 ⑬postWaitForIdle(retryable, failedAttempts + 1);
 						else ⑧当前线程是工作线程，需要先切换到主线程，再从主线程切换到 HandlerThread 线程
  							--> ⑨AndroidWatchExecutor#postWaitForIdle
                          	   		--> ⑩mainHandler#post//通过主线程 Handler 切换到主线程执行	
                          	   			==> ⑪AndroidWatchExecutor#waitForIdle//等待空闲时发送
                          	   			
```

可以看到 `Retryable#run` 在工作线程执行。`AndroidWatchExecutor` 内创建了一个 HandlerThread ，通过它创建了一个 Handler。如果当前线程是主线程，直接调用主线程的  `Message#addIdleHandler` 进行处理，给工作线程发送延时任务。否则，需要先调用 `mainHandler#post` 方法， 切换到主线程，然后再调用 `AndroidWatchExecutor#waitForIdle`。之所以要切换到主线程，主要是为了使用主线程的 `MessageQueue#addIdleHandler`

```java
HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
handlerThread.start();
backgroundHandler = new Handler(handlerThread.getLooper());
```



前面所谈都是执行线程的切换，下面开始内存分析任务具体是如何进行的。

### 2.3内存泄漏分析

#### 2.3.1工作机制

1.  在后台线程检查引用是否被清除，如果没有，调用 GC。
2.  如果引用还是未被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 `.hprof` 文件中。
3.  在另外一个进程中的 `HeapAnalyzerService` 有一个 `HeapAnalyzer` 使用[HAHA](https://github.com/square/haha) 解析这个文件。
4.  得益于唯一的 reference key, `HeapAnalyzer` 找到 `KeyedWeakReference`，定位内存泄漏。
5.  `HeapAnalyzer` 计算 *到 GC roots 的最短强引用路径*，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链。
6.  引用链传递到 APP 进程中的 `DisplayLeakService`， 并以通知的形式展示出来。

#### 2.3.2RefWatcher#ensureGone

在前面的线程切换到目标工作线程之后就调用该方法。

```java
RefWatcher#ensureGone
	--> ① RefWatcher#removeWeaklyReachableReferences//移除所有「弱可达引用」
	if (debuggerControl.isDebuggerAttached())
      	--> return RETRY;//调试模式控制
	if (gone(reference)) //判断类名是否不在 Set<String> retainedKeys 中，若不在，说明内存分析完毕
    	  return DONE; 
    --> gcTrigger.runGc();//触发 gc，调用的是  Runtime.getRuntime().gc();
	--> ② RefWatcher#removeWeaklyReachableReferences//再次移除所有「弱可达引用」
	if !gone(reference)//如果该引用对应的键仍然在 retainKeys 中，说明可能存在内存泄漏，进行 dump 然后分析
      	--> File heapDumpFile = heapDumper.dumpHeap();// dump 堆内存，会触发 stop the world
	    if heapDumpFile == RETRY_LATER //现在无法触发 gc ，先返回。稍候重试
            --> return RETRY;
        --> new HeapDump
        --> ServiceHeapDumpListener#analyze
        	--> HeapAnalyzerService#runAnalysis
        		--> startService //开启 HeapAnalyzerService 服务，需要 HeapDump 以及接收分析结果的回调类的类名
```



#### 2.3.3RefWatcher#removeWeaklyReachableReferences

移除在 retainedKeys 中的所有弱可达的对象对应的 key。

```java
private final Set<String> retainedKeys;
ReferenceQueue<Object> queue;
private void removeWeaklyReachableReferences() {
  // 对象一旦只存在弱引用，马上就会被加入 ReferenceQueue 中，这发生在 finalization 或者 gc 之前
  KeyedWeakReference ref;
  while ((ref = (KeyedWeakReference) queue.poll()) != null) {
    retainedKeys.remove(ref.key);//移除弱引用对应的 key
  }
}
```

#### 2.3.4KeyedWeakReference

上面使用到了一个 KeyedWeakReference 类型的对象，KeyedWeakReference 意为「带键的弱引用」。具体实现如下：

```java
final class KeyedWeakReference extends WeakReference<Object> {
  public final String key;
  public final String name;

  KeyedWeakReference(Object referent, String key, String name,
      ReferenceQueue<Object> referenceQueue) {
    super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
    this.key = checkNotNull(key, "key");
    this.name = checkNotNull(name, "name");
  }
}
```



唯一创建了 KeyedReference 的地方：

`RefWatcher#watch(Object, String)`

```java
public void watch(Object watchedReference, String referenceName) {
  if (this == DISABLED) {
    return;
  }
  checkNotNull(watchedReference, "watchedReference");
  checkNotNull(referenceName, "referenceName");
  final long watchStartNanoTime = System.nanoTime();
  String key = UUID.randomUUID().toString();//创建一个独一无二 key
  retainedKeys.add(key);//添加到 retainedKeys 中 
  final KeyedWeakReference reference =
      new KeyedWeakReference(watchedReference, key, referenceName, queue);//创建一个 「带键的弱引用」

  ensureGoneAsync(watchStartNanoTime, reference);
}
```

#### 2.3.5retainedKeys

`retainedKeys` 类型为一个 `Set<String> `，主要用于判断对象是否被回收。

retainedKeys 的添加与删除

##### 添加：

一个带键的弱引用（KeyedWeakReference）被创建的时候，其中的键会被加入 retainedKeys 中。

##### 删除：

通过 RefWatcher#removeWeaklyReachableReferences 方法 可以删除 retainedKeys 中那些仅含有弱引用的 KeyedWeakReference 对应的 key。

如果不在 retainedKeys 中说明该对象已经被回收了。



#### 2.3.6HeapAnalyzerService

HeapAnalyzerService 是一个 IntentService，它会在  onHandleIntent 方法中对堆进行分析。

HeapAnalyzerService#onHandleIntent

```java
@Override protected void onHandleIntent(Intent intent) {
  if (intent == null) {// intent 为空直接返回
    CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
    return;
  }
  String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);//获取回调类的类名
  HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);//获取 HeapDump 

  HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);//创建 HeapAnalyzer

  AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);//检查泄漏（通过 HAHA 来完成），并获取结果
  AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);//将分析结果发送给监听器
}
```

可以看到，onHandleIntent 方法中，会从 Intent 中获取 HeapDump 以及 listenerClassName（监听器类名），然后将相应的数据交给 HAHA 库中的 HeapAnalyzer 进行分析，最后将结果发回给监听器

注：在创建对分析器的时候 会使用到 heapDump.excludedRefs，excludedRefs  实际类型为 AndroidExcludedRefs ，它是一个枚举类，其中设置了一些由于特定制造商的实现引起内存泄漏的类。如果内存泄漏是由该枚举类含有的类锁引起，那么内存泄漏问题会被忽略。

对于大部分应用开发者而言都应该使用 createAppDefaults 方法，不过也可以自己创建一个 EnumSet ，在其中指定自己要忽略的内存泄漏类，并通过 `AndroidExcludedRefs#createBuilder(EnumSet)`方法进行设置。

#### 2.3.7HeapAnalyzer#checkForLeak 检测内存泄漏

限于篇幅，仅对 HAHA 进行简单介绍

HAHA 是一个 Java 库，可以自动完成对 Android 堆转储文件的分析。

这个项目实际上是对其他人的工作的重新打包，以使其成为一个小型的 Maven 依赖项。



HAHA  可达性分析算法 

```java
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
  long analysisStartNanoTime = System.nanoTime();

  if (!heapDumpFile.exists()) {
    Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
    return failure(exception, since(analysisStartNanoTime));
  }

  try {
    HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);//通过堆转储文件构造一个 MemoryMappedFileBuffer
    HprofParser parser = new HprofParser(buffer);//创建一个 Hprof 文件解析器
    Snapshot snapshot = parser.parse();//解析，将结果赋给 Snapshot 
    deduplicateGcRoots(snapshot);//删除重复的 GC Root 

    Instance leakingRef = findLeakingReference(referenceKey, snapshot);//利用 referenceKey 和  snapshot 寻找发生泄漏的引用

    // False alarm, weak reference was cleared in between key check and heap dump.
    if (leakingRef == null) {
      return noLeak(since(analysisStartNanoTime));//没有发生内存泄漏
    }

    return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);//发现内存泄漏
  } catch (Throwable e) {
    return failure(e, since(analysisStartNanoTime));//
  }
}
```



1.  得益于唯一的 reference key, `HeapAnalyzer` 找到 `KeyedWeakReference`，定位内存泄漏。
2.  `HeapAnalyzer` 计算 *到 GC roots 的最短强引用路径*，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链。



## 三、总结

内存泄漏的本质是长生命周期的对象持有短生命周期对象的强引用，导致短生命周期对象使用完了之后无法被回收。也就是应该被回收的对象没有被回收。
那么问题就变成了什么对象是应该被回收的对象呢？对于 Activity 而言，执行完 onDestroy 方法之后，就是应该被回收了。因此可以将 Activity#ondestroy 方法作为一个检测点。Application 中提供了各个 Activity 的生命周期回调方法的监听，LeakCanary 就是通过注册  `ActivityLifecycleCallbacks` ，监听生命周期方法的回调，作为整个内存泄漏分析的入口。

每次 `onActivityDestroyed(Activity activity) ` 方法被回调之后,都会创建一个 `KeyedWeakReference` 对相应 Activity 的状态进行跟踪，手动调用 gc，后台线程（HandlerThread ）检查引用是否被清除，如果没有就手动调用一次 gc，如果这时还是没有被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 `.hprof` 文件中。在另一个进程中的 `HeapAnalyzerService`  中，  HeapAnalyzer 会通过 haha 开源库对文件进行分析。	得益于唯一的 reference key,  `HeapAnalyzer`  找到 `KeyedWeakReference`，定位内存泄漏。`HeapAnalyzer` 计算 *到 GC roots 的**最短强引用路径***，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链。引用链传递到 APP 进程中的 `DisplayLeakService`， 并以通知的形式展示出来。



### 进阶使用：

了解 LeakCanary 的原理之后，发现其实它就是在对象不可用的时候去判断对象是否被回收了，但 LeakCanary 只检查了 Activity，我们是否可以检查其他对象呢，毕竟 Activity 泄漏只是内存泄漏的一种，答案当然是可以的，我们只要需要进行如下操作:

```java
LeakCanary.install(app).watch(object)
```

但我们在调用这个方法的时候**需要确定这个 object 已经不需要了，可以被回收了**。通过这种方式我们就可以**对任何对象都进行检测**了。



## 四、参考资料与学习资源推荐

-   [LeakCanary 原理浅析](www.jianshu.com/p/3f1a1cc1e964)
-   [LeakCanary 中文使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)
-   [话说 ReferenceQueue](https://hongjiang.info/java-referencequeue/)
-   [Reference 、ReferenceQueue 详解](https://www.jianshu.com/p/f86d3a43eec5)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源。谢谢！

