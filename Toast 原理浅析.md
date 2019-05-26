---
title: Toast 原理
date: 2017-07-28 14:51:12
tags: 
- 原理分析
categories:
- 原理分析
---

## 序言

`Toast.makeText(context,"msg",Toast.Length_SHORT).show();`

我们都知道调用了这行代码，便会触发显示 Toast 信息，但是从调用开始到显示出来的具体过程是怎么样的，里面具体实现又是怎么样的呢？

<!--more-->

## 原理

在 Toast 内部有两类 IPC 过程。
- 第一类： Toast 访问 NotificationManagerService
- 第二类：NotificationManagerService 回调 Toast 里的 TN 接口。


Toast 属于系统 Window，它内部的视图由两种方式指定，一种是系统默认的样式，另一种是自定义一个 view 然后通过 setView 方法将 view 设置进去。
不管哪一种方式，它们都对应 Toast 的一个 View 类型的内部成员 mNextView。

Toast 提供 show 和 cancel 方法分别用于显示和隐藏 view，它们的内部都是一个 IPC 过程。

Toast 有定时取消功能，所以系统采用了 Handler（利用其中的 sendMessageDelayed() 方法）


Toast.show() 调用流程大致如下：

![toast show](https://user-images.githubusercontent.com/16668676/28705462-7393612a-73a2-11e7-92b0-5d0ebfb237f0.jpg)





先来看看 Toast.makeText 方法
```
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
    Toast result = new Toast(context);//创建一个新的 Toast 对象

    LayoutInflater inflate = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);//获取 LayoutInflater
    View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null); //渲染出系统默认的布局
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    tv.setText(text);//将我们的信息设置到 TextView 中去
    
    result.mNextView = v;//把 view 赋给 Toast 内部的View
    result.mDuration = duration;//设置 toast 时长

    return result;
}
```
再瞧一瞧 Toast.show(); 方法
```
public void show() {
    if (mNextView == null) {//判断，如果没有设置 Toast 的 view，就抛出异常
        throw new RuntimeException("setView must have been called");
    }

    INotificationManager service = getService();//获取 INotificationManager 
    String pkg = mContext.getOpPackageName();// 获取调用者的包名
    TN tn = mTN;//给 TN 赋值
    tn.mNextView = mNextView;

    try {
        service.enqueueToast(pkg, tn, mDuration);
    } catch (RemoteException e) {
        // Empty
    }
}
```
这里面有几个地方看起来比较陌生 INotificationManager 是什么，TN 又是什么？


INotificationManager 的值是从 getService() 方法中得来的。我们先查看下 getService() 的内部实现：
```
static private INotificationManager getService() {
    if (sService != null) {
        return sService;
    }
    sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
    return sService;
}
```
- 了解 Binder 的同学应该一看便知道，这里用到了 Binder。
- INotificationManager 的具体实现在 NotificationManagerService 中，简便起见，后续内容将 NotificationManagerService 称为 NMS。

TN 又是什么？
```java
private static class TN extends ITransientNotification.Stub {
    final Runnable mHide = new Runnable() {
        @Override
        public void run() {
            handleHide();
            // Don't do this in handleHide() because it is also invoked by handleShow()
            mNextView = null;
        }
    };

    private final WindowManager.LayoutParams mParams = new WindowManager.LayoutParams();
    final Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            IBinder token = (IBinder) msg.obj;
            handleShow(token);
        }
    };

    int mGravity;
    int mX, mY;
    float mHorizontalMargin;
    float mVerticalMargin;


    View mView;
    View mNextView;
    int mDuration;

    WindowManager mWM;

    static final long SHORT_DURATION_TIMEOUT = 5000;
    static final long LONG_DURATION_TIMEOUT = 1000;

    TN() {
        // XXX This should be changed to use a Dialog, with a Theme.Toast
        // defined that sets up the layout params appropriately.
        final WindowManager.LayoutParams params = mParams;
        params.height = WindowManager.LayoutParams.WRAP_CONTENT;
        params.width = WindowManager.LayoutParams.WRAP_CONTENT;
        params.format = PixelFormat.TRANSLUCENT;
        params.windowAnimations = com.android.internal.R.style.Animation_Toast;
        params.type = WindowManager.LayoutParams.TYPE_TOAST;
        params.setTitle("Toast");
        params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
    }

    /**
     * schedule handleShow into the right thread
     */
    @Override
    public void show(IBinder windowToken) {
        if (localLOGV) Log.v(TAG, "SHOW: " + this);
        mHandler.obtainMessage(0, windowToken).sendToTarget();
    }

    /**
     * schedule handleHide into the right thread
     */
    @Override
    public void hide() {
        if (localLOGV) Log.v(TAG, "HIDE: " + this);
        mHandler.post(mHide);
    }

    public void handleShow(IBinder windowToken) {
        if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                + " mNextView=" + mNextView);
        if (mView != mNextView) {
            // remove the old view if necessary
            handleHide();
            mView = mNextView;
            Context context = mView.getContext().getApplicationContext();
            String packageName = mView.getContext().getOpPackageName();
            if (context == null) {
                context = mView.getContext();
            }
            mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
            // We can resolve the Gravity here by using the Locale for getting
            // the layout direction
            final Configuration config = mView.getContext().getResources().getConfiguration();
            final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
            mParams.gravity = gravity;
            if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                mParams.horizontalWeight = 1.0f;
            }
            if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                mParams.verticalWeight = 1.0f;
            }
            mParams.x = mX;
            mParams.y = mY;
            mParams.verticalMargin = mVerticalMargin;
            mParams.horizontalMargin = mHorizontalMargin;
            mParams.packageName = packageName;
            mParams.hideTimeoutMilliseconds = mDuration ==
                Toast.LENGTH_LONG ? LONG_DURATION_TIMEOUT : SHORT_DURATION_TIMEOUT;
            mParams.token = windowToken;
            if (mView.getParent() != null) {
                if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                mWM.removeView(mView);
            }
            if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
            mWM.addView(mView, mParams);
            trySendAccessibilityEvent();
        }
    }

    private void trySendAccessibilityEvent() {
        AccessibilityManager accessibilityManager =
                AccessibilityManager.getInstance(mView.getContext());
        if (!accessibilityManager.isEnabled()) {
            return;
        }
        // treat toasts as notifications since they are used to
        // announce a transient piece of information to the user
        AccessibilityEvent event = AccessibilityEvent.obtain(
                AccessibilityEvent.TYPE_NOTIFICATION_STATE_CHANGED);
        event.setClassName(getClass().getName());
        event.setPackageName(mView.getContext().getPackageName());
        mView.dispatchPopulateAccessibilityEvent(event);
        accessibilityManager.sendAccessibilityEvent(event);
    }        

    public void handleHide() {
        if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
        if (mView != null) {
            // note: checking parent() just to make sure the view has
            // been added...  i have seen cases where we get here when
            // the view isn't yet added, so let's try not to crash.
            if (mView.getParent() != null) {
                if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                mWM.removeViewImmediate(mView);
            }

            mView = null;
        }
    }
}
```
TN 是 Toast 的一个私有的静态内部类，继承自 ITransientNotification.Stub 
- 也是用到了 Binder 机制。

在回到 show 方法。该方法最后调用了 `service.enqueueToast(pkg, tn, mDuration);` 方法。我们到 NMS 看看该方法的主要实现。

```java
 @Override
public void enqueueToast(String pkg, ITransientNotification callback, int duration)
{

    //....
    final boolean isSystemToast = isCallerSystem() || ("android".equals(pkg));//是否是 android 系统的 toast
    final boolean isPackageSuspended =
            isPackageSuspendedForUser(pkg, Binder.getCallingUid());
    //...
    synchronized (mToastQueue) {
        int callingPid = Binder.getCallingPid();
        long callingId = Binder.clearCallingIdentity();
        try {
            ToastRecord record;
            int index = indexOfToastLocked(pkg, callback);//根据包名和回调来获取的对应包名的 Toast 的在 mToastQueue 中的位置（如果有的话）
            //如果对应包名的 Toast 已经存在了，则直接在原位更新，不会把它排到队尾
            if (index >= 0) {
                record = mToastQueue.get(index);
                record.update(duration);
            } else {
                //限制非系统应用的数量（不能超过 50），防止 DOS 攻击同时也是为了解决内存泄漏问题
                if (!isSystemToast) {
                    int count = 0;
                    final int N = mToastQueue.size();
                    for (int i=0; i<N; i++) {
                         final ToastRecord r = mToastQueue.get(i);
                         if (r.pkg.equals(pkg)) {
                             count++;
                             if (count >= MAX_PACKAGE_NOTIFICATIONS) {
                                 Slog.e(TAG, "Package has already posted " + count
                                        + " toasts. Not showing more. Package=" + pkg);
                                 return;
                             }
                         }
                    }
                }

                Binder token = new Binder();
                mWindowManagerInternal.addWindowToken(token,
                        WindowManager.LayoutParams.TYPE_TOAST);
                record = new ToastRecord(callingPid, pkg, callback, duration, token);//将 Toast 包装为 ToastRecord
                mToastQueue.add(record);//加入 mToastQueue
                index = mToastQueue.size() - 1;
                keepProcessAliveIfNeededLocked(callingPid);
            }
            if (index == 0) {
                showNextToastLocked();//显示下一条 Toast
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
}
```
该方法会将 Toast 请求封装为 ToastRecord 并将其添加进 mToastQueue 队列中。
- mToastQueue 是一个 ArrayList
- 注意：每个非系统应用最多只能有 50 个ToastRecord。如果超出了最大值，就会出错。
    - 这样做主要是为了 防止 DOS（Denial Of Service）

> 拒绝服务攻击（英语：denial-of-service attack，缩写：DoS attack、DoS）亦称洪水攻击，是一种网络攻击手法，其目的在于使目标电脑的网络或系统资源耗尽，使服务暂时中断或停止，导致其正常用户无法访问。

将 ToastRecord 加入队列之后， `enqueueToast` 还调用了 `showNextToastLocked();` 方法, 该方法的具体实现如下：
```
void showNextToastLocked() {
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
        try {
            record.callback.show(record.token);//调用 record 对象中 callback 的 show 方法
            scheduleTimeoutLocked(record); //超时提醒，控制显示时间
            return;
        } catch (RemoteException e) {
            //...代码省略
        }
    }
}
```
这里的 callBack 是什么？
- 它是在 ToastRecord 中的，瞅瞅 ToastRecord 的构造方法。
```java
ToastRecord(int pid, String pkg, ITransientNotification callback, int duration,
            Binder token) {
    this.pid = pid;
    this.pkg = pkg;
    this.callback = callback;
    this.duration = duration;
    this.token = token;
}
```
在 enqueueToast 方法中，将 Toast 包装为 ToastRecord ，创建了 ToastRecord 对象。
- `record = new ToastRecord(callingPid, pkg, callback, duration, token);//将 Toast 包装为 ToastRecord` 
- callBack 是 enqueueToast 中的一个参数，我们的调用如下： `service.enqueueToast(pkg, tn, mDuration);` 
- 没错，callBack 实际上就是前面讲到的 Toast 的内部类 TN。
    - 回到前面看看，TN 确实继承了 `ITransientNotification.Stub`。

- showNextToastLocked() 方法调用 callBack 的 show() 方法来显示 Toast。
```
@Override
public void show(IBinder windowToken) {
    if (localLOGV) Log.v(TAG, "SHOW: " + this);
    mHandler.obtainMessage(0, windowToken).sendToTarget();
}

```


```
final Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        IBinder token = (IBinder) msg.obj;
        handleShow(token);
    }
};

```
其具体实现又是在 handleShow(token);
```java
public void handleShow(IBinder windowToken) {
 
    if (mView != mNextView) {
        // 如果有必要的话，将还在显示的 toast 隐藏掉
        handleHide();
        mView = mNextView;
        //代码省略
        mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);//获取 windowManager
        //省略代码，给布局参数赋值
        if (mView.getParent() != null) {
            if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeView(mView);
        }
        mWM.addView(mView, mParams);
        trySendAccessibilityEvent();
    }
}
```
以上代码核心在于
```java
mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
mWM.addView(mView, mParams);
```
将 Toast 的视图添加到 Window 中。 这样 View 就能顺利显示出来了。

你可能会问，为什么 TN 调用 show 方法要用到的 Handler ？
- 因为该方法是被是被 NMS 以跨进程方式调用的，因此它们运行在 Binder 线程池中。为了将执行环境切换到 Toast 请求所在的线程，在它们内部使用了 Handler。


那么时间到了 Toast 又是怎么样取消的呢？
- 在令 Toast 显示方法调用过程中 我们也调用了 `scheduleTimeoutLocked(record);` 方法。
```
private void scheduleTimeoutLocked(ToastRecord r){
    mHandler.removeCallbacksAndMessages(r);
    Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
    long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;//设置 延迟时间
    mHandler.sendMessageDelayed(m, delay);
}
```
- SHORT_DELAY 为 2s
- LONG_DELAY 为 3.5s

scheduleTimeoutLocked 会发出消息给 hanlder,收到相应的信息后 handleMessage 方法会调用`handleTimeout((ToastRecord)msg.obj);`, 该方法又会调用 `cancelToastLocked(index);`

```java
void cancelToastLocked(int index) {
    ToastRecord record = mToastQueue.get(index);
    try {
        record.callback.hide();//
    } catch (RemoteException e) {
        //代码省略
    }
        //代码省略
}
```
可以看到隐藏 Toast 的实现也是在 TN 中的。与 handleShow 对应，有一个 handleHide 方法。
该方法会将 Toast 的视图从 Window 中移除。如下所示：
```java
public void handleHide() {
    if (mView != null) {
        if (mView.getParent() != null) {
            mWM.removeViewImmediate(mView);
        }
        mView = null;
    }
}
```





## Toast 的「一个 Bug」

其实这是一个系统的Bug，谷歌为了让应用的 Toast 能够显示在其他应用上面，所以使用了通知栏相关的 API，但是这个 API 随着用户屏蔽通知栏而变得不可用，系统错误地认为你没有通知栏权限，从而间接导致 Toast 有 show 请求时被系统所拦截



##  参考资料与学习资源推荐

- [Toast通知栏权限填坑指南](https://www.jianshu.com/p/1d64a5ccbc7c)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！