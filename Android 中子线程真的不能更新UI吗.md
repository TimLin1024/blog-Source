---
title: Android 中子线程真的不能更新 UI 吗？
date: 2017-08-16 17:06:31
tags: 
- 原理分析
- 探究
categories: 
- 原理分析
- 探究
---

# Android 中子线程真的不能更新 UI 吗？

2020-04-18  更新。

先说结论：Android 中子线程在满足一定的条件下可以更新 UI。

<!--more-->

## 一个栗子：

```java
public class MainActivity extends AppCompatActivity {

    private ImageView mImageView;
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      	mImageView = (ImageView)findViewById(R.id.iv);
        new Thread(new Runnable() {
            @Override
            public void run() {
				mImageView.setImageResource(R.drawable.ic_book);//更新 ui
            }
        }).start();
    }
}
```

如上在 onCreate 方法中新建一个线程对 mImageView 进行了操作，成功从子线程更新了 ui。

但是如果让线程 sleep 一段时间（比如 300ms），

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            Thread.sleep(300);//睡眠 300 ms
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        mImageView.setImageResource(R.drawable.ic_book);//更新 ui
    }
}).start();
```

那么就很可能会报如下错误：(如果 300ms 不报错，可将其改为 1000ms)

```java
  android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
      at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:7194)
      at android.view.ViewRootImpl.invalidateChildInParent(ViewRootImpl.java:1111)
      at android.view.ViewGroup.invalidateChild(ViewGroup.java:4833)
      at android.view.View.invalidateInternal(View.java:12102)
      at android.view.View.invalidate(View.java:12062)
      at android.view.View.invalidate(View.java:12046)
      at android.widget.ImageView.setImageDrawable(ImageView.java:456)
      at android.support.v7.widget.AppCompatImageView.setImageDrawable(AppCompatImageView.java:100)
      at android.support.v7.widget.AppCompatImageHelper.setImageResource(AppCompatImageHelper.java:89)
      at android.support.v7.widget.AppCompatImageView.setImageResource(AppCompatImageView.java:94)
      at com.android.rdc.librarysystem.MainActivity$1.run(MainActivity.java:52)
      at java.lang.Thread.run(Thread.java:818)
```



## 分析

该异常是从哪里抛出的？   

从出错的堆栈信息中可以异常看到是 `ViewRootImpl#checkThread()` 方法中抛出的。

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

当访问 UI 时，ViewRootImpl 会调用 checkThread 方法去检查当前访问 UI 的线程是否为创建 UI 的那个线程，如果不是。则会抛出异常。但是**为什么一开始在 MainActivity 的 onCreate 方法中创建一个子线程访问 UI，程序还是正常能跑起来呢**？    

>   上述例子中的 Thread 执行时，ViewRootImpl 还没创建，ViewRootImpl 无法对 view tree 的根节点 DecorView 执行 performTraversals，view tree 里的所有 View 都没有被赋值 mAttachInfo（注：AttachInfo 中存储了一组信息。当 View 被连接到它的父节点时，会给这个 View 的 AttachInfo 赋值）。

>   在 onCreate 完成时，Activity 并没有完成初始化 view tree。**view tree 的初始化是从 ViewRootImpl 执行 performTraversals 开始**，这个过程会对 view tree 进行从根节点 DecorView 开始的遍历，对所有视图完成初始化，初始化包括视图的大小布局，以及 AttachInfo，ViewParent 等属性域的初始化。

`ImageView#setImageResource  `触发的调用流程 

```java
ImageView#setImageResource 
  -->  如果最新资源的宽度或者高度跟已有的不同，
  	--> View#requestLayout
  		--> 满足条件，最终会调用 ViewRootImpl#requestLayout
	-->  View#invalidate 
		-->  View#invalidate(boolean)
      		-->  View#invalidateInternal //如果 
  			if mAttachInfo 以及 mParent 都不为空
      			--> ViewGroup#invalidateChild
      				//这里会不断循环去取上一个结点的 mParent,一直到 mParent == null 也就是到达顶部 View 为止
                        -->  ViewRootImpl#invalidateChildInParent // 注意 DecorView 的 mParent 是 ViewRootImpl
                            -->  ViewRootImpl#checkThread //在这里执行 checkThread，如果当前线程不是创建 UI 的线程则抛出异常
             else
        
----------------------------------------------------------------------
//View#invalidateInternal 

final AttachInfo ai = mAttachInfo;
final ViewParent p = mParent;
//只有当 mAttachInfo 以及 mParent 都不为空时，才会触发重绘
if (p != null && ai != null && l < r && t < b) {
    //....
    p.invalidateChild(this, damage);
}
```

从上述流程可以看出，只有在 mAttachInfo 以及 mParent 都不为空时， `ViewGroup#invalidateChild` 才会被调用，该方法最终会触发 checkThread，而向上面所提到的， onCreate 方法调用时 ViewRootImpl 还未创建， mAttachInfo 以及 mParent 均为 null，所以在子线程修改 UI 不会报错。

但是这个时候对 View 的修改是有效果的。那么，ViewRootImpl 创建之前的，程序对 UI 的更新操作是如何进行的呢？在最初的`ImageView#setImageResource  ` 方法中已经将要图片资源 id 赋给了ImageView 的一个属性 mResource ，等到 ViewRootImpl  创建完毕之后就可以得到更新了。

### ViewRootImpl 何时被创建？

回过头看，抛异常的方法既然是 ViewRootImpl 中的方法，那首先应该去看看 ViewRootImpl 是在哪里、在什么时候被创建的。

如果你对 Activity 的启动流程有所了解，应该知道，Activity 的启动与生命周期都是由 ActivityThread 相应的方法触发的。我们知道每一个 Activity 都有一个顶级 View ——DecorView，当 Activity 中的视图显示出来的时候 DecorView 肯定已经创建完毕了。而 ViewRootImpl 作为 DecorView 与 WindowManager 之间的「桥梁」，应该也是在视图变得可见之前被创建出来的。说到视图可见与否，一般都会想起 onResume（实际上 onResume 调用时，Activity 的视图也不一定可见）。

从`ActivityThread#handleLaunchActivity` 方法出发，查看其调用流程

```java
ActivityThread#handleLaunchActivity
	--> performLaunchActivity //创建 Activity 
    *--> handleResumeActivity
        --> performResumeActivity//回调 onResume
            -->  Activity#performResume();
                --> Instrumentation#callActivityOnResume
                    --> Activity#onResume();//回调 onResume 
      **--> Activity#makeVisible();
            --> WindowManagerGlobal#addView()
                --> root = new ViewRootImpl(view.getContext(), display);//创建 ViewRootImpl
                --> ViewRootImpl#setView
```

从上述流程可以看出，ViewRootImpl 是在 WindowManagerGlobal#addView() 方法中被创建出来的。并且是在 Activity#onResume 方法调用之后才被创建。因此我们如果在 onResume 方法中创建一个子线程去修改 UI，大多数情况下也是可以成功的。



## 一个在子线程更新 UI 的栗子：

创建一个 handlerThread并调用它的 start 方法，获取handlerThread 中的 looper 构造一个  Handler。在该 Handler的 handleMessage方法（运行在子线程） 中将 view 添加到 WindowManger里面，并支持进行更新操作。

[示例代码地址](https://github.com/TimLin-pro/UpdateUiOnSubThread/blob/master/app/src/main/java/com/timlin/updateuionsubthread/ManageUiOnHandlerThreadActivity.java)

## 总结：

ViewRootImpl 的创建在 onResume 方法回调之后，而我们一开篇是在 onCreate 方法中创建了子线程并访问 UI，在那个时刻，ViewRootImpl 还没有创建，我们在子线程调用 了 ImageView#setImageResource，虽然可能会触发 View#requestLayout 和 View#invalidate() ，但是由于 ViewRootImpl还未创建出来，因此 ViewRootImpl#checkThread 没有被调用到，也就是说，检测当前线程是否是创建的 UI 那个线程 的逻辑没有执行到，所以程序没有崩溃一样能跑起来。而之后修改了程序，让线程休眠了 300 毫秒后，程序就崩了。很明显 300 毫秒后 ViewRootImpl 已经创建了，可以执行 checkThread 方法检查当前线程。

开篇的例子中我们在 onCreate 方法中创建的子线程访问 UI 是一种极端的情况。实际开发中不会这么做。



#### 下次如果有人问你 Android 中子线程真的不能更新 UI 吗？ 你可以这么回答：

任何线程都可以更新自己创建的 UI。只要保证满足下面几个条件就好了

- 在 ViewRootImpl 还没创建出来之前

  - UI 修改的操作没有线程限制。

- 在 ViewRootImpl 创建完成之后

  1. 保证「创建 ViewRootImpl 的操作」和「执行修改 UI 的操作」在同一个线程即可。也就是说，要在同一个线程调用 ViewManager#addView 和 ViewManager#updateViewLayout 的方法。
     - 注：ViewManager 是一个接口，WindowManger 接口继承了这个接口，我们通常都是通过 WindowManger（具体实现为 WindowMangerImpl） 进行 view 的 add remove update 操作的。

  2. 对应的线程需要创建 Looper 并且调用 Looper#loop 方法，开启消息循环。



#### 有同学可能会问，保证上述条件 1 成立，不就可以避免 checkThread 时候抛出异常了吗？为什么还需要开启消息循坏？

- 条件 1 可以避免检查异常，但是无法保证 UI 可以被绘制出来。
- 条件 2 可以让更新的 UI 效果呈现出来
  - WindowManger#addView 最终会调用 WindowManageGlobal#addView 方法，进而触发ViewRootImpl#setView 方法，该方法内部会调用 ViewRootImpl#requestLayout 方法。
  - 了解过 UI 绘制原理的同学应该知道 下一步就是 scheduleTraversals 了，该方法会往消息队列中插入一条消息屏障，然后调用 Choreographer#postCallback 方法，往 looper 中插入一条异步的 MSG_DO_SCHEDULE_CALLBACK 消息。等待垂直同步信号回来之后执行。
    - 注：ViewRootImpl 有一个 Choreographer  成员变量，ViewRootImpl 的构造函数中会调用 Choreographer#getInstance(); 方法，获取一个当前线程的 Choreographer 局部实例。



#### 使用子线程更新 UI 有实际应用场景吗？

Android 中的  SurfaceView 通常会通过一个子线程来进行页面的刷新。如果我们的自定义 View 需要频繁刷新，或者刷新时数据处理量比较大，那么可以考虑使用 SurfaceView 来取代 View。



## 参考资料与学习资源推荐

-   [Android 中子线程真的不能更新 UI 吗？](http://blog.csdn.net/xyh269/article/details/52728861)
-   [多线程学习之--真的不能在子线程里更新 UI 吗？](http://blog.csdn.net/u010198148/article/details/51779567)
-   [互联网笔记 Android 中子线程真的不能更新 UI 吗？](https://www.zybuluo.com/natsumi/note/736165)
-   [Android 只在 UI 主线程修改 UI，是个谎言吗？ 为什么这段代码能完美运行？](https://www.zhihu.com/question/24764972/answer/36053366)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源。谢谢！