---
title: Android源码中的代理模式
date: 2017-08-18 17:45:11
tags: 
- 原理分析
- Android 进阶
- 设计模式
categories: 
- 原理分析
- Android 进阶
- 设计模式
---



##  代理模式的定义 
代理模式也称为委托模式，为其他对象提供一种代理以控制对这个对象的访问。


## 代理模式的使用场景 
当无法或不想访问某个对象或者访问某个对象存在困难时可以**通过一个代理对象来间接访问**。

为了保证客户端使用的透明性，委托对象与代理对象需要实现相同的接口（或继承相同的抽象类）

<!--more-->


## 代理模式的UML类图 
![proxy uml](https://user-images.githubusercontent.com/16668676/29449202-6aed8b00-842c-11e7-8e3f-362ead423f2d.png)

角色介绍：
- Subject 抽象主题类
    - 主要职责是声明真实主题类与代理的共同接口方法，该类可以是一个抽象类也可以是一个接口
- RealSubject 真实主题类
    - 也称为被委托类或被代理理类，该类定义了代理所表示的真实对象，由其执行具体的业务逻辑方法，而客户类则通过代理类间接地调用真实主题类中定义的方法。
- ProxySubject 代理类
    - 也称为委托类，持有一个对真实主题类的引用，在其实现的接口方法中调用真实主题类中相应的接口方法执行，以此起到代理的作用。
- Client 使用代理类的类


## 代理模式的简单实现 
代理模式大致可分为两大部分，**静态代理和动态代理**。

**动态代理**通过反射机制动态地生成代理者的对象，code 阶段不需要知道代理谁，代理的真正对象会在执行阶段确定。

Java 提供一个便捷的动态代理接口 InvocationHandler，实现该接口需要重写其调用方法 invoke。

动态代理通过一个代理类来代理 N 多个被代理类，其实质是对代理者与被代理者进行解耦，使两者之间没有直接的的耦合关系。

代理可以看作是 对调用目标的一个包装，这样我们对目标代码的调用不是直接发生的，而是代理完成。（观点：大部分动态代理的场景可以看作是 装饰器模式的应用。）

### 静态代理 vs 动态代理
相对而言静态代理则只能为给定接口下的实现类做代理，如果接口不同那么就需要重新定义不同代理类，较为复杂。
- 但是静态代理更符合面向对象的原则。

实际开发中具体使用哪种方式来实现代理，看自己的偏好。


### 分类
静态代理和动态代理是从 code 方面来区分代理模式的。

也可以**从其使用范围来区分**不同类型的代理实现：
- **远程代理**（Remote Proxy）为某个对象**在不同的内存地址空间**提供局部代理。使系统可以将 Server 部分的实现隐藏，以便 Client 可以不考虑 Server 的存在。
- **虚拟代理**（Virtual Proxy）使用一个代理对象**表示一个十分耗资源的对象并在真正需要时才创建**。
- **保护代理**(Protection Proxy)：使用代理**控制对原始对象的访问**。该类型的代理常被**用于原始对象具有不同访问权限**的情况。
- **智能引用**(Smart Reference)：在访问原始对象时执行一些自己的**附加操作并对指向原始对象的引用计数**。

静动态代理都可以应用于上述 4 种情形。


## Android源码中的代理模式实现 
以 ActivityManager 为例。

![source code proxy](https://user-images.githubusercontent.com/16668676/29449446-6771e5ce-842d-11e7-935f-d1e5685d43af.png)

- 抽象接口: IActivityManager
- 代理类 ActivityManagerProxy
- 被代理类
    - ActivityManagerNative(抽象类，并不处理太多的逻辑，大部分逻辑由其子类——ActivityManagerService 承担)
    - ActivityManagerService(真实部分)

---
- ActivityManagerService 是系统级的 Service 并且运行于独立的进程空间中，可以通过 ServiceManger 获取到它。
- ActivityManagerProxy 运行于自己所处的进程空间中（也就是说，与 ActivityManagerService 运行在不同的进程中）
- 所以此处源码所实现的代理实质为==远程代理==。

ActivityManagerProxy 在实际的逻辑处理并**没有过多地被外部类**使用，Android 中管理与维护 Activity 相关信息的类是另一个叫做 ActivityManager 的类。不过实质上 ActivityManager 大多数逻辑还是由 ActivityManagerProxy 承担。（注意：ActivityManager 并没有实现 **I**ActivityManager 接口，它直接继承自 Object）

以 ActivityManager 的 getAppTasks() 方法为例
```Java
public List<ActivityManager.AppTask> getAppTasks() {
    ArrayList<AppTask> tasks = new ArrayList<AppTask>();
    List<IAppTask> appTasks;
    try {
        appTasks = ActivityManagerNative.getDefault().getAppTasks(mContext.getPackageName());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
    int numAppTasks = appTasks.size();
    for (int i = 0; i < numAppTasks; i++) {
        tasks.add(new AppTask(appTasks.get(i)));
    }
    return tasks;
}
```

`ActivityManagerNative.getDefault(); `方法 返回一个 `IActivityManager` 类型的对象，通过该对象调用其 getAppTasks 方法
```Java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```
gDefault 到底是什么？
```Java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");//获取 AMS
        //代码省略
        IActivityManager am = asInterface(b);//完成创建一个 AMS 的客户端端代理对象—— ActivityManagerProxy
        //代码省略
        return am;
    }
};
```
上述代码中构造了一个 `Singleton<IActivityManager>` 类型的 gDefault 对象，其中通过 `ServiceManager.getService("activity");` 获取一个系统级的 Service，而该 Service 实质上就是 AMS，接着调用 asInterface 方法，创建一个 ActivityManagerProxy 对象。



ActivityManagerNative.asInterface 方法的具体实现
```
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }

    return new ActivityManagerProxy(obj);
}
```

ActivityManagerProxy 的 `getTasks` 方法，将数据打包跨进程传递给 Server 端的 AMS 处理
```Java
public List<ActivityManager.RunningTaskInfo> getTasks(int maxNum, int flags)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeInt(maxNum);
        data.writeInt(flags);
        mRemote.transact(GET_TASKS_TRANSACTION, data, reply, 0);
        reply.readException();
        ArrayList<ActivityManager.RunningTaskInfo> list = null;
        int N = reply.readInt();
        if (N >= 0) {
            list = new ArrayList<>();
            while (N > 0) {
                ActivityManager.RunningTaskInfo info =
                        ActivityManager.RunningTaskInfo.CREATOR
                                .createFromParcel(reply);
                list.add(info);
                N--;
            }
        }
        data.recycle();
        reply.recycle();
        return list;
    }
```

看看 AMS 中的 getTasks 方法的具体实现。
```java
@Override
public List<IAppTask> getAppTasks(String callingPackage) {
    int callingUid = Binder.getCallingUid();
    long ident = Binder.clearCallingIdentity();

    synchronized(this) {
        ArrayList<IAppTask> list = new ArrayList<IAppTask>();
        try {
            if (DEBUG_ALL) Slog.v(TAG, "getAppTasks");

            final int N = mRecentTasks.size();
            for (int i = 0; i < N; i++) {
                TaskRecord tr = mRecentTasks.get(i);
                // Skip tasks that do not match the caller.  We don't need to verify
                // callingPackage, because we are also limiting to callingUid and know
                // that will limit to the correct security sandbox.
                if (tr.effectiveUid != callingUid) {
                    continue;
                }
                Intent intent = tr.getBaseIntent();
                if (intent == null ||
                        !callingPackage.equals(intent.getComponent().getPackageName())) {
                    continue;
                }
                ActivityManager.RecentTaskInfo taskInfo =
                        createRecentTaskInfoFromTaskRecord(tr);
                AppTaskImpl taskImpl = new AppTaskImpl(taskInfo.persistentId, callingUid);
                list.add(taskImpl);
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        return list;
    }
}
```



## Android 中的 Binder 跨进程通信机制与 AIDL 
四个重要类：
- Binder Client 类比 PC、终端设备
- Binder Server 类比 服务器
- Binder Driver（实现在内核中） 类比 路由器
- Binder Manager 类比 DNS 服务器

因为依赖于抽象，加上本地代理类的存在，所以调用远程方法时，还是与调用本地的普通的方法一样。但是代理类内部需要对参数什么的进行打包处理，然后再通过 Binder 机制，跨进程将数据传递给远程方法，远程方法执行完毕再返回数据。

![binder 通信大致模型图](https://user-images.githubusercontent.com/16668676/29453380-021cc0ea-843c-11e7-901d-a7490d12f9ea.jpg)



Binder Driver 实现在内核中，主要负责 Binder 通信的建立，以及 Binder数据在进程间的传递和 Binder 引用计数管理/数据包的传输等。

Binder Client 和 Binder Server 之间的跨进程通信统一通过 Binder Driver 处理转发，
- 对于 Binder Client 而言，它只需要知道自己要使用的 Binder 的名字以及该 **Binder 实体**在 ServerManager 中的 0 号引用即可。
    - 访问原理：
        - 通过 0 号引用去访问 ServerManager **获取该 Binder 的引用**，
        - 得到引用后就可以像普通方法调用那样调用 Binder 实体的方法
- ServerManager 用来管理 Binder Server（Android 中通常是一个 Service）
    - Binder Client 通过它来查询 Binder Server 的引用
    - ServerManager **是一个标准的 Binder Server**，并且在 Android 中约定其在 Binder 通信过程中的唯一标识（类似于 IP 地址）永远为 0。
        - 在 IServiceManager 接口中定义有对外开放的接口方法，供客户端访问。
- **匿名 Binder**。很多时候 Binder Server 会将一个 Binder 实体封装进数据包传递给 Binder Client，而此时 Binder Server 会在该数据包中标注 Binder 实体的位置，Binder Driver 会为该匿名的 Binder 生成实体结点和实体引用，并将该引用传递给 Binder Client。


IServiceManager 接口(声明 ServerManger 对外提供的方法、数据 )
```java
public interface IServiceManager extends IInterface {

    public IBinder getService(String name) throws RemoteException;
    
    public IBinder checkService(String name) throws RemoteException;

     */
    public void addService(String name, IBinder service, boolean allowIsolated)
                throws RemoteException;
    public String[] listServices() throws RemoteException;

    public void setPermissionController(IPermissionController controller)
            throws RemoteException;
    
    static final String descriptor = "android.os.IServiceManager";

    int GET_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION;
    int CHECK_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+1;
    int ADD_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+2;
    int LIST_SERVICES_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+3;
    int CHECK_SERVICES_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+4;
    int SET_PERMISSION_CONTROLLER_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+5;
}
```





## 参考

《Andorid 源码设计模式解析与实战》

如果本文中有不正确的结论、说法，请大家提出和我讨论，共同进步，谢谢！




