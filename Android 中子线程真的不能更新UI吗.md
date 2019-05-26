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

下面代码仅用于演示，不适合直接用在实际项目中。

创建一个 handlerThread并调用它的 start 方法，获取handlerThread 中的 looper 构造一个  Handler。在该 Handler的 handleMessage方法（运行在子线程） 中将 view 添加到 WindowManger里面，并支持进行更新操作。

```java
package com.rdc.timlin.appiumdemo;

import android.graphics.Color;
import android.graphics.PixelFormat;
import android.graphics.drawable.ColorDrawable;
import android.os.Bundle;
import android.os.Handler;
import android.os.HandlerThread;
import android.os.Message;
import android.support.v7.app.AppCompatActivity;
import android.view.Gravity;
import android.view.View;
import android.view.WindowManager;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {
    public static final int CREATE_VIEW = 1;
    public static final int UPDATE_VIEW = 2;
    private Handler mHandler;
    private TextView mTextView;
    private WindowManager mWindowManager;
    private Button mBtnCreateView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mWindowManager = getWindowManager();

        HandlerThread handlerThread = new HandlerThread("my handler thread");
        handlerThread.start();
        mTextView = new TextView(MainActivity.this);
        mHandler = new Handler(handlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams();
                layoutParams.format = PixelFormat.TRANSPARENT;//设置为 透明，默认效果是 黑色的
                layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;//设置window透传，也就是当前view所在的window不阻碍底层的window获得触摸事件。
                switch (msg.what) {
                    case CREATE_VIEW:
                        mTextView.setText("created at non-ui-thread");
                        mTextView.setBackground(new ColorDrawable(Color.WHITE));
                        layoutParams.width = WindowManager.LayoutParams.WRAP_CONTENT;
                        layoutParams.height = WindowManager.LayoutParams.WRAP_CONTENT;
                        layoutParams.gravity = Gravity.CENTER;
                        mWindowManager.addView(mTextView, layoutParams);
                        mBtnCreateView.setClickable(false);//添加 TextView 不能 add 两次。add 完之后就屏蔽点击事件
                        break;
                    case UPDATE_VIEW:
                        mTextView.setBackground(new ColorDrawable(Color.WHITE));
                        mTextView.setText("updated at non-ui-thread");
                        layoutParams.width = WindowManager.LayoutParams.WRAP_CONTENT;
                        layoutParams.height = WindowManager.LayoutParams.WRAP_CONTENT;
                        layoutParams.gravity = Gravity.LEFT;
                        mWindowManager.updateViewLayout(mTextView, layoutParams);
                        break;
                }
            }
        };
        initView();
    }

    private void initView() {
        mBtnCreateView = findViewById(R.id.btn_create_view);
        Button btnUpdateView = findViewById(R.id.btn_update_view);
        mBtnCreateView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mHandler.sendEmptyMessage(CREATE_VIEW);
            }
        });

        btnUpdateView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mHandler.sendEmptyMessage(UPDATE_VIEW);
            }
        });
    }
}
```

布局文件。activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/btn_create_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Create View in subThread"/>

    <Button
        android:id="@+id/btn_update_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="update View in subThread"/>

</LinearLayout>
```

## 总结：

ViewRootImpl 的创建在 onResume 方法回调之后，而我们一开篇是在 onCreate 方法中创建了子线程并访问 UI，在那个时刻，ViewRootImpl 还没有创建，无法检测当前线程是否是创建的 UI 那个线程，所以程序没有崩溃一样能跑起来，而之后修改了程序，让线程休眠了 300 毫秒后，程序就崩了。很明显 300 毫秒后 ViewRootImpl 已经创建了，可以执行 checkThread 方法检查当前线程。

开篇的例子中我们在 onCreate 方法中创建的子线程访问 UI 是一种极端的情况。实际开发中不会这么做。



下次如果有人问你 Android 中子线程真的不能更新 UI 吗？ 你可以这么回答：

> 子线程可以更新 UI，但是需要创建子线程的根视图（ViewRoot），并添加到 WindowManager，还要创建子线程的 Looper。以上条件都满足时，它可以修改它自己创建的根视图中的 UI。



## 参考资料与学习资源推荐

-   [Android 中子线程真的不能更新 UI 吗？](http://blog.csdn.net/xyh269/article/details/52728861)
-   [多线程学习之--真的不能在子线程里更新 UI 吗？](http://blog.csdn.net/u010198148/article/details/51779567)
-   [互联网笔记 Android 中子线程真的不能更新 UI 吗？](https://www.zybuluo.com/natsumi/note/736165)
-   [Android 只在 UI 主线程修改 UI，是个谎言吗？ 为什么这段代码能完美运行？](https://www.zhihu.com/question/24764972/answer/36053366)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源。谢谢！