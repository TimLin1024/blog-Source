---
title: Android 源码中的装饰者模式
date: 2017-09-05 15:54:40
tags: 
- 原理分析
- Android 进阶
- 设计模式
categories:
- 原理分析
- Android 进阶
- 设计模式
---

## 装饰者模式的定义

装饰模式（Decorator Pattern） 也称**包装模式**（Wrapper Pattern），是结构型设计模式之一，使用一种对客户端透明的方式来**动态地拓展对象的功能**，同时它也是继承关系的一种替代方案之一。

通过装饰者模式可以动态地给一个对象添加一些额外的职责。就增加功能而言，装饰模式相比生成子类更灵活。因为它装饰者持有一个被装饰者的引用，因此可以方便地调用具体被装饰者对象中的方法，因此可以在不破坏原类层次结构的情况下为类增加一些功能，我们只需要在被装饰者对象的相应方法前后增加相应的功能逻辑即可。

<!--more-->

## 使用场景

需要透明且动态地拓展类的功能时。

## UML 类图

![decorator uml](https://user-images.githubusercontent.com/16668676/30046081-a4423c2c-923a-11e7-9dad-170e000c5ece.png)



-   Component  抽象组件
    -   可以是接口或者抽象类，充当一个被装饰的原始对象。（在该模式中位于继承结构的顶部，大家都直接/间接地继承它）
-   ConcreteComponent 组件具体实现类
    -   该类是 Component 类的具体实现，也是我们装饰的具体对象。
-   Decorator  抽象装饰者
    -   继承自 Component 并且**必须持有一个指向 Component 的引用**
    -   通常会在其方法中调用 ConcreteComponent 的方法。
    -   如果装饰逻辑单一，可以直接省略该类，直接写一个具体的装饰者对象即可。
-   ConcreteDecoratorA,B,C
    -   继承自 Decoration， 在父类对 Component 的方法调用基础上上，增加自己的一些功能。（通常都是在基础方法执行前或者后调用自己新增的方法）

使用时经常会把 ComponentImpl 或者说 ConcreteComponent 传入给具体的 Decorator。

```java
Compoent component = new ConponentImpl();
DecoratorImpl decoratorImpl = new DecoratorImpl(component);
decoratorImpl.xxOperation();
```





## Android 源码中的模式实现



![context](https://user-images.githubusercontent.com/16668676/30047048-b4aeb6e2-9241-11e7-9a04-b11bb1acdc44.png)

角色简介：

-   Context ：抽象组件
-   ComtextImpl ：Context 的具体实现类
-   ContextWrapper ：装饰者的父类（其中的所有方法都只是调用了 ContextImpl 中对应的方法）
-   ContextThemeWrapper ：继承自 ContextWrapper  的装饰者
-   Activity ：继承自 ContextThemeWrapper 的装饰者

我们以常用的方法为例，看看装饰者模式在其中的具体实现方式

```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }

   //启动 Activity
   @Override
    public void startActivity(Intent intent, Bundle options) {
        mBase.startActivity(intent, options);
    }
  	
    //发送广播
    @SystemApi
    @Override
    public void sendBroadcast(Intent intent, String receiverPermission, Bundle options) {
        mBase.sendBroadcast(intent, receiverPermission, options);
    }
	
    //注册监听器
      @Override
    public Intent registerReceiver(
        BroadcastReceiver receiver, IntentFilter filter) {
        return mBase.registerReceiver(receiver, filter);
    }
}
```

从以上的代码中可以看出，ContextWrapper 作为装饰者的父类，持有 Context 的引用 mBase（mBase 的实际类型为 ContextImpl），其中的所有方法都只是调用了 ContextImpl 中对应的方法。



```java
class ContextImpl extends Context {
     //启动 Activity 的逻辑实现
    @Override
    public void startActivity(Intent intent) {
        warnIfCallingFromSystemProcess();
        startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // 代码省略
        //调用 Instrumentation.execStartActivity() 方法
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
  
    @Override
    public void sendBroadcast(Intent intent, String receiverPermission) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        String[] receiverPermissions = receiverPermission == null ? null
                : new String[] {receiverPermission};
        try {
            intent.prepareToLeaveProcess(this);
          	//调用 AMS 的 broadcastIntent 方法
            ActivityManagerNative.getDefault().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, receiverPermissions, AppOpsManager.OP_NONE,
                    null, false, false, getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    
    //注册广播的方法
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }  
   //注册广播
  	@Override
  	public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
   //注册广播的具体实现
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
          	//调用 AMS 的 registerReceiver 方法
            final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
  
}
```

可以看到，ContextImpl 中提供了具体的方法实现。



ContextWrapper 的子类，例如 Activity 会根据需要对具体方法的实现进行装饰或者修改。

比如 startActivity() 方法，Activity 没有使用被装饰者的实现，而是自己实现了一套逻辑。

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback {
        
        @Override
        public void startActivity(Intent intent) {
            this.startActivity(intent, null);
        }
        
         @Override
        public void startActivity(Intent intent, @Nullable Bundle options) {
            if (options != null) {
                startActivityForResult(intent, -1, options);
            } else {
                // Note we want to go through this call for compatibility with
                // applications that may have overridden the method.
                startActivityForResult(intent, -1);
            }
        }   
             
       public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,@Nullable Bundle options) {
              if (mParent == null) {
                  options = transferSpringboardActivityOptions(options);
                  //调用 Instrumentation.execStartActivity() 方法
                  Instrumentation.ActivityResult ar =
                      mInstrumentation.execStartActivity(
                          this, mMainThread.getApplicationThread(), mToken, this,
                          intent, requestCode, options);
                //代码省略
        }
}
```

## ContextImpl 的创建

从上面解析中我们知道 Context 的实现中使用了装饰者模式也知道了 ContextImpl 是 Context 具体实现类，但是 ContextImpl 是在上面地方被初始化的呢？

因为 Activity 启动之后我们便可以调用 Context 中的方法了，我们猜想 ContextImpl 是在 Activity 创建过程中初始化的。

对 Android Framework 层有所了解同学应该知道 Activity 是由 AMS 管理的，AMS 会通过调用 ApplicationThread 与间接地控制 Activity。 ApplicationThread 的 scheduleXxx 方法中会调用 sendMessage 方法将相应的 Message 发送给 H，H 根据不同的 Message 调用 ActivityThread 中相应的 handleXxx 方法。

ActivityThread#H

```java
private class H extends Handler {
	//代码省略
    public void handleMessage(Message msg) {
        //代码省略
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
              	//调用 handleLaunchActivity
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;       
    //代码省略
}
```

当需要启动新的 Activity 时，ApplicationThread 的  scheduleLaunchActivity 方法会先被调用，该方法会通过 H 调用 handleLaunchActivity 方法，而 handleLaunchActivity 方法又会调用  performLaunchActivity 方法。

ActivityThread#performLaunchActivity 

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

  //代码省略
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
      	//创建 Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
          	//获取 Context
            Context appContext = createBaseContextForActivity(r, activity);
			//将前面准备的值关联到 Activity 中
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window);
        //代码省略
    }
    return activity;
}
```

ActivityThread#createBaseContextForActivity

```java
private Context createBaseContextForActivity(ActivityClientRecord r, final Activity activity) {
    ContextImpl appContext = ContextImpl.createActivityContext(//调用 ContextImpl 的静态方法创建 Activity Context
            this, r.packageInfo, r.token, displayId, r.overrideConfig);
    appContext.setOuterContext(activity);//外部的 context（此处为 Activity）设置给 ContextImpl
    Context baseContext = appContext;
	//代码省略
    return baseContext;
}
```



ContextImpl#createActivityContext()

```java
static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, IBinder activityToken, int displayId,
        Configuration overrideConfiguration) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    return new ContextImpl(null, mainThread, packageInfo, activityToken, null, 0,
            null, overrideConfiguration, displayId);
}
```

经过上面的分析，我们可以得出结论，Activity 的 Context 是在 performLaunchActivity 方法中通过调用 createBaseContextForActivity  初始化的。在 createBaseContextForActivity 方法中，通过调用 ContextImpl 的静态方法 createActivityContext 创建 获取一个 ContextImpl 的实例对象，并通过 setOuterContext 方法将两者建立关联。

## 总结

学过代理模式的同学可能觉得装饰模式与代理模式有点像（因为同样持有引用）。但是既然是它们是两个不同的设计模式，先看看它们各自的定义。

装饰模式：以对客户端透明的方式**扩展对象的功能**，是**继承关系的一个替代方案**；
代理模式：给一个对象提供一个代理对象，并**由代理对象来控制对原有对象的引用**；

光看定义可能还是比较模糊。二者**区别**在哪里呢？

-   装饰模式应该为所装饰的对象**增强功能**
-   代理模式对代理的对象施加控制，但**不对对象本身的功能进行增强**。

可以简单地理解为：你在一个地方写装饰，大家就知道这是在增加功能，你写代理，大家就知道是在限制。

## 参考资料与学习资源推荐

-   《Android 源码设计模式解析与实战》