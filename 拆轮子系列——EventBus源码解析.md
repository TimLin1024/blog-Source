---
title: 拆轮子系列——EventBus 源码解析
date: 2017-10-14 12:09:21
tags: 
- 源码分析
- 框架原理
categories: 
- 源码分析
- 框架原理
---



项目地址：[EventBus](https://link.jianshu.com/?t=https://github.com/greenrobot/EventBus)，本文分析版本: 3.1.1

## 一、概述

EventBus 是一个 Android **事件发布/订阅框架**，通过**解耦**发布者和订阅者简化 Android 事件传递，这里的事件可以理解为消息，本文中统一称为事件。事件传递既可用于 Android 四大组件间通讯，也可以用户异步线程和主线程间通讯等等。
传统的事件传递方式包括：Handler、BroadCastReceiver、Interface 回调，相比之下 EventBus 的优点是代码简洁，使用简单，并将事件发布和订阅充分解耦。

<!--more-->



- **事件(Event)：**又可称为消息，本文中统一用事件表示。其实就是一个对象，可以是网络请求返回的字符串，也可以是某个开关状态等等。`事件类型(EventType)`指事件所属的 Class。
  - 事件分为一般事件和 Sticky 事件，相对于一般事件，Sticky 事件不同之处在于，当事件发布后，再有订阅者开始订阅该类型事件，依然能收到该类型事件**最近一个** Sticky 事件（所谓「最近一个」指的就是该类型事件「最后一次发出」）。
- **订阅者(Subscriber)：**订阅某种事件类型的对象。当有发布者发布这类事件后，EventBus 会执行订阅者的 onEvent 函数，这个函数叫`事件响应函数`。订阅者通过 register 接口订阅某个事件类型，unregister 接口退订。订阅者存在优先级，优先级高的订阅者可以取消事件继续向优先级低的订阅者分发，默认所有订阅者优先级都为 0。
- **发布者(Publisher)：**发布某事件的对象，通过 post 接口发布事件。

## 二、如何使用

### 2.1 添加依赖

#### 方式一，运行期处理注解

在app 的 build.gradle 文件中添加依赖

```Groovy
dependencies {
    implementation 'org.greenrobot:eventbus:3.1.1'
}
```

#### 方式二，编译期预处理注解

Android Studio  3.0 及以上

在 app 的 build.gradle 文件中添加

```groovy
android {
	//……
    defaultConfig {
        //……
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [eventBusIndex: 'org.greenrobot.eventbusperf.MyEventBusIndex']
            }
        }
    }
}

dependencies {
    implementation 'org.greenrobot:eventbus:3.1.1'
    annotationProcessor  'org.greenrobot:eventbus-annotation-processor:3.0.1'
}
```



build 之后，会生成一个 MyEventBusIndex.java类。

然后在使用 EventBus 实例之前，又有两种方式可以将配置MyEventBusIndex.java配置到类中是哟经。



###### 方式一 在构造 EventBus 时传入我们自定义的 EventBusIndex，

```java
EventBus mEventBus = EventBus.builder().addIndex(new MyEventBusIndex()).build();
```

###### 方式二 将索引应用到默认的单例中

使用 EventBus 之前，先调用下面的代码初始化 EventBus。

```java
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
```


### 2.2 定义事件类

```java
public class CustomEvent {
    private String mEventName;

    public CustomEvent() {
    }

    public String getEventName() {
        return mEventName;
    }

    public void setEventName(String eventName) {
        mEventName = eventName;
    }
}
```



### 2.3 注册为监听者

在合适的地方（比如 Activity#onCreate、Fragment#onCreateView）通过下方代码进行注册

`EventBus.getDefault().register(this);`

### 2.4 编写响应事件的订阅方法

```java
@Subscribe(threadMode = ThreadMode.BACKGROUND, sticky = true, priority = 100)
public void onMessage(CustomEvent event) {
    Log.d(TAG, "onMessage: " + event.getEventName());
}
```

使用编译期注解处理的情况下，订阅方法的访问控制权限必须是 非 private 并且非 static 的

使用运行期反射处理的情况下，订阅方法的访问控制权限必须是 public 的

- 通过 ThreadMode 可以指定订阅方法在哪个线程执行，有四种选择
  - ThreadMode.MAIN 事件订阅方法会在 UI 线程中执行
    - 使用此模式的事件订阅方法必须快速返回以避免阻塞主线程。
  - ThreadMode.POSTING （默认的模式）表示事件在哪个线程中发布出来的，事件订阅方法就会在这个线程中运行；
    - 该模式避免了线程切换，适用于那些在很短的时间内完成的简单任务，无需主线程。使用这种模式的事件订阅方法必须快速返回以避免阻塞发布线程（发布线程可能是主线程）。
  - ThreadMode.MAIN_ORDERED
    - 在 Android 上，订阅方法将在 Android 的主线程中被调用。事件将会排队等待交付，这确保了 post 调用是非阻塞的。
  - ThreadMode.BACKGROUND 子线程执行，如果本来就在子线程，直接在该子线程执行
    - EventBus 使用一个后台线程，将按顺序发送所有事件。使用这种模式的订阅方法应该尽快返回以避免阻塞后台线程。
      - 注意：「一个后台线程」所指的并不是 `Executors.newSingleThreadPool()`，而是使用 EventBus 在实例化时创建的 cacheThreadPool 中的某一个线程。
  - ThreadMode.ASYNC 新建子线程执行。适用于耗时操作
    - 发布事件永远不会等待使用此模式的订阅方法。适用于比较耗时的订阅方法，比如用于网络请求。使用时应该避免同时触发大量长时间运行的异步订阅方法来限制并发线程的数量。 EventBus 使用线程池有效地重用已完成的异步用户通知中的线程。
- 通过 sticky 指定是否接收粘性事件，默认为 false
- 通过 priority 设置接收订阅方法的优先级，相同的事件，优先级越高的订阅方法 越早收到事件

### 2.5 发送事件

通过`EventBus`的`post()`方法来发送事件, 发送之后就会执行注册过这个事件的对应类的方法. 或者通过`postSticky()`来发送一个粘性事件。

### 2.6 解除注册

在合适的地方（比如 Activity#onDestroy）使用下面的代码进行解除注册 `EventBus.getDefault().unregister(this);`



### 2.7 小结

要实现订阅，需要进行注册，以及解注册，订阅方法以「目标事件」作为方法的参数， 使用 Subscribe 注解，可以指定订阅方法执行的线程、是否接收 sticky 事件、订阅方法的优先级。

至于发送方，只需要创建相应的 事件实例，然后调用 post 或者 postSticky 将事件发送出去即可。

## 三、实现

### 3.1 初始化 EventBus

开发者通常是调用 EventBus#getDefault 方法获取 EventBus 实例。

```java
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```

getDefault 通过双重校验锁的方式来实现单例

#### 构造方法

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

public EventBus() {
    this(DEFAULT_BUILDER);
}
```

通过 getDefault 获取的 EventBus 对象是通过默认的 EventBusBuilder 构造而成的。

```java
private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();//默认为 CachedThreadPool，不限制线程数

boolean logSubscriberExceptions = true;
boolean logNoSubscriberMessages = true;
boolean sendSubscriberExceptionEvent = true;
boolean sendNoSubscriberEvent = true;
boolean throwSubscriberException;

boolean strictMethodVerification;//
ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;//默认的线程池
List<Class<?>> skipMethodVerificationForClasses;
List<SubscriberInfoIndex> subscriberInfoIndexes;
MainThreadSupport mainThreadSupport;

EventBus(EventBusBuilder builder) {
	//……
    subscriptionsByEventType = new HashMap<>();
    typesBySubscriber = new HashMap<>();
    stickyEvents = new ConcurrentHashMap<>();
	//……
    eventInheritance = builder.eventInheritance;
    executorService = builder.executorService;
}
```

主要看以下几个单例的实现。

```java
boolean eventInheritance = true;//是否允许事件继承
boolean ignoreGeneratedIndex;//是否忽略 生成的 index，默认为 false，也就是会先尝试寻找编译期注解生成的订阅方法信息，找不到再使用反射去获取。

private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
private final Map<Object, List<Class<?>>> typesBySubscriber;
private final Map<Class<?>, Object> stickyEvents;

private final HandlerPoster mainThreadPoster;
private final BackgroundPoster backgroundPoster;
private final AsyncPoster asyncPoster;

```

- subscriptionsByEventType ，key 是事件类型，value 为 订阅了该事件的方法列表
- typesBySubscriber，key 为订阅者，value 某个订阅者订阅的事件列表
- stickyEvents，key 为事件类型，value 为具体的事件实例
- mainThreadPoster 主线程分发
- backgroundPoster 后台线程分发
- asyncPoster 异步线程分发

### 3.2 注册订阅

org.greenrobot.eventbus.EventBus#register

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();//获取订阅者的 class 对象
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);//查找订阅者中所有的订阅方法
    synchronized (this) {
        //迭代遍历订阅者中所有的订阅方法
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```



EventBus#findSubscriberMethods 

找出给定 class 中所有的订阅方法

```java
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap();//以 class 为 key，方法列表为 value 的，Map 作为缓存

List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List subscriberMethods = (List)METHOD_CACHE.get(subscriberClass);//缓存中获取
    if(subscriberMethods != null) {
        return subscriberMethods;//缓存命中，直接返回
    } else {
        if(this.ignoreGeneratedIndex) {//忽略编译期生成的 订阅方法信息
            subscriberMethods = this.findUsingReflection(subscriberClass);//通过反射获取订阅方法信息
        } else {
            //获取编译期生成的 订阅方法信息
            subscriberMethods = this.findUsingInfo(subscriberClass);//
        }

        if(subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);//添加到缓存中
            return subscriberMethods;//返回
        }
    }
}
```



EventBus#subscribe()

```java
//必须从同步块中调用
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;//事件类型
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);//新建一个 Subscription，存储订阅的对象以及 响应的方法
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);//Map<Class<?>, CopyOnWriteArrayList<Subscription>> 从 map 中获取相应订阅类型的 列表
    if (subscriptions == null) {//如果没有则新建一个
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);//没有则抛出异常。
        }
    }

    int size = subscriptions.size();
    //遍历监听列表，将新的 subscription 插入到正确位置。列表按照优先级递减的顺序排序
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
	//如果触发的方法 要接收「粘性事件」，获取相应类型的 Event 并触发相应的方法
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

该方法会将相应的事件插入对应事件的列表中。如果在方法注解中声明了 sticky，还会马上调用该方法。

检测 stick 事件，如果相应的事件定义有子类的话，会遍历事件的事件子类逐一通知该方法。



#### 3.2.1 通过反射处理注解

SubscriberMethodFinder#findUsingReflection

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findUsingReflectionInSingleClass(findState);
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

SubscriberMethodFinder#findUsingReflectionInSingleClass

```java
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();//获取的是类的所有公有方法，这就包括自身 和从基类继承的、从接口实现的所有 public 方法。
            //getDeclareMethods 返回的是该类中定义的「所有方法」，但是不包括从父类继承而来的方法
        } catch (Throwable th) {
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {//遍历方法
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {//参数列表长度为 0
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //将订阅方法信息添加到 findState 中
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```





##### ignoreGeneratedIndex 是什么?

由于反射成本高,而且 EventBus 3.0 引入了 EventBusAnnotationProcessor,故默认 ignoreGeneratedIndex 为 false,**需要注意的是,如果设置 ignoreGeneratedIndex 为 true,则前面使用的 MyEventBusIndex 无效,还是会走反射解析的分支**。

#### 3.2.2 使用编译期生成的订阅方法信息

网上有很多介绍 EventBus 的文章,但是几乎没有提到 EventBusAnnotationProcessor 的。在 3.0 版本开始，`EventBus`提供了一个`EventBusAnnotationProcessor`注解处理器来在编译期通过读取`@Subscribe()`注解并解析,处理其中所包含的信息,然后生成`java`类来保存所有订阅者关于订阅的信息,这样就比在运行时使用反射来获得这些订阅者的信息速度要快.我们可以参考`EventBus`项目里的[EventBusPerformance](https://link.jianshu.com?t=https://github.com/greenrobot/EventBus/tree/master/EventBusPerformance)这个例子,编译后我们可以在`build`文件夹里找到这个类,[MyEventBusIndex](https://link.jianshu.com?t=https://github.com/greenrobot/EventBus/blob/master/EventBusPerformance/build.gradle#L27) 类,当然类名是可以自定义的.我们大致看一下生成的`MyEventBusIndex`类是什么样的:



订阅者中的 订阅方法

```java
public class ReceiveEventFragment extends Fragment {
    //……
    
    @Subscribe(threadMode = ThreadMode.MAIN, priority = 0)
    public void onReceive(MsgEvent event) {
        Log.d(TAG, "onReceive: " + event);
        mTvMsg.setText(String.format("msgId:%d\nmsg:%s", event.msgId, event.msg));
    }

    @Subscribe(threadMode = ThreadMode.MAIN, priority = 10000)
    public void ShowToast(MsgEvent event) {
        Toast.makeText(getActivity(), String.format("msgId:%d\nmsg:%s", event.msgId, event.msg), Toast.LENGTH_SHORT).show();
    }
}
```

```java
/** This class is generated by EventBus, do not edit. */
public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();//以订阅者为 key，以订阅者中的 订阅方法列表为 value 的 map
		//将 ReceiveEventFragment 的订阅信息存储到 map 中
        putIndex(new SimpleSubscriberInfo(com.test.commentdemo.eventbusdemo.ReceiveEventFragment.class,
                true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onReceive", com.test.commentdemo.eventbusdemo.MsgEvent.class,
                    ThreadMode.MAIN),
            new SubscriberMethodInfo("ShowToast", com.test.commentdemo.eventbusdemo.MsgEvent.class,
                    ThreadMode.MAIN, 10000, false),
        }));//订阅者中所有的订阅方法
        
        //……代码省略（存储其他订阅者的订阅信息到 map 中）
    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);//根据类型从 Map 获取订阅者的订阅信息
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```





继续前面 ignoreGeneratedIndex 为 false 时，会执行以下分支。

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();//获取订阅的方法信息
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);//如果找不到相应的订阅方法信息（可能使用 EventBus 实例之前，没有将MyEventBusIndex ），需要通过反射获取订阅方法信息
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}

```

`findUsingInfo()`方法,其无非就是通过查找我们前面所说的`MyEventBusIndex`类中的信息,来转换成`List<SubscriberMethod>`从而获得订阅类的相关订阅函数的各种信息



SubscriberMethodFinder#getSubscriberInfo

```java
private SubscriberInfo getSubscriberInfo(SubscriberMethodFinder.FindState findState) {
    if(findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) 
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();//获取父类的订阅方法信息
        if(findState.clazz == superclassInfo.getSubscriberClass()) {//
            return superclassInfo;
        }
    }

    if(this.subscriberInfoIndexes != null) {
        Iterator superclassInfo1 = this.subscriberInfoIndexes.iterator();
		//遍历，从 index 中获取 订阅信息
        while(superclassInfo1.hasNext()) {
            SubscriberInfoIndex index = (SubscriberInfoIndex)superclassInfo1.next();
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);//从自动生成的 MyEventBusIndex 类中的 SUBSCRIBER_INDEX 里面获取订阅方法信息
            if(info != null) {
                return info;
            }
        }
    }
    return null;
}
```



### 3.3 解注册

```java
/** Unregisters the given subscriber from all event classes. */
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);//获取订阅者订阅的事件列表
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {//遍历订阅者所订阅的事件
            //从事件对应的订阅列表中 订阅者的订阅信息
            unsubscribeByEventType(subscriber, eventType);
        typesBySubscriber.remove(subscriber);//移除订阅者
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

```java
/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    //从事件对应的订阅列表中 订阅者的订阅信息
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```



### 3.4 发送事件

EventBus#post

#### 1. 获取订阅列表

```java
public class EventBus {

    private final ThreadLocal<EventBus.PostingThreadState> currentPostingThreadState;//线程私有变量——当前发布线程状态

    public void post(Object event) {
        EventBus.PostingThreadState postingState = (EventBus.PostingThreadState)this.currentPostingThreadState.get();//记录发布线程的状态（比如是否是主线程）
        List eventQueue = postingState.eventQueue;//从发布状态中获取事件队列
        eventQueue.add(event);//添加进事件队列的队尾
        if(!postingState.isPosting) {//当前不是处于分发状态
            postingState.isMainThread = this.isMainThread();//是否在主线程
            postingState.isPosting = true;//将状态改为发布中
            if(postingState.canceled) {//取消发布
                throw new EventBusException("Internal error. Abort state was not reset
            }

            try {
                while(!eventQueue.isEmpty()) {//只要事件队列非空，就一直往外取出事件并发布
                    this.postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
                                            
    static final class PostingThreadState {
        final List<Object> eventQueue = new ArrayList();//事件队列
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;

        PostingThreadState() {
        }
    }                                                                   
}
```



```java
final static class PostingThreadState {
    final List<Object> eventQueue = new ArrayList<>();//事件队列
    boolean isPosting;//是否正在分发中
    boolean isMainThread;//是否在主线程
    Subscription subscription;
    Object event;//事件
    boolean canceled;//取消
}

private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};//currentPostingThreadState 是一个线程局部变量


/** Posts the given event to the event bus. */
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();//获取当前线程事件分发状态
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);//添加到事务队列中

    if (!postingState.isPosting) {//如果当前不是在分发状态,则进入分发状态
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            //编译事件队列，逐一进行分发
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```



EventBus#postSingleEvent

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {//如果事件支持子类型，查找该事件的所有子类型
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

EventBus#postSingleEventForEventType，       

```java
 //查找 事件所对应的订阅者列表，然后迭代列表，切换到目标线程执行
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);//查找 事件所对应的订阅者列表
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {//事件取消了
                break;
            }
        }
        return true;
    }
    return false;
}
```





#### 2.切换到指定线程

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING://
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND://后台线程
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC://新建一个子线程处理
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

##### BACKGROUND

```java
public void enqueue(Subscription subscription, Object event) {
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);//包装为一个 PendingPost
    synchronized (this) {//加锁，进入同步块
        queue.enqueue(pendingPost);//添加到队列中
        if (!executorRunning) {//如果现在没有在执行 后台任务，则获取一个新线程执行任务
            executorRunning = true;
            eventBus.getExecutorService().execute(this);
        }
    }
}

@Override
public void run() {
    try {
        try {
            while (true) {//循环，直到队列为空
                PendingPost pendingPost = queue.poll(1000);//获取 PendingPost，最多等到 1 秒

                if (pendingPost == null) {
	                //前面取不到 PendingPost，下面进行双重校验检查
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();//
                        if (pendingPost == null) {
                            //队列确实为空，停止运行
                            executorRunning = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);//通过反射调用相应的方法
            }
        } catch (InterruptedException e) {
            eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
        }
    } finally {
        executorRunning = false;
    }
}
```

##### MAIN

```java
mainThreadPoster.enqueue(subscription, event);
```



HandlerPoster#enqueue

```java
public void enqueue(Subscription subscription, Object event) {
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
    synchronized (this) {//加锁
        queue.enqueue(pendingPost);//加入 pendingQueue 中。该队列会在 handleMessage 方法中调用
        if (!handlerActive) {//如果当前 handler 不是处于活跃状态，则退出
            handlerActive = true;
            if (!sendMessage(obtainMessage())) {//发送信息，提示
                throw new EventBusException("Could not send handler message");
            }
        }
    }
}
```

org.greenrobot.eventbus.HandlerPoster#handleMessage

```java
@Override
public void handleMessage(Message msg) {
    boolean rescheduled = false;
    try {
        long started = SystemClock.uptimeMillis();
        while (true) {
            PendingPost pendingPost = queue.poll();
            if (pendingPost == null) {
                synchronized (this) {
                    // Check again, this time in synchronized
                    pendingPost = queue.poll();
                    if (pendingPost == null) {
                        handlerActive = false;
                        return;
                    }
                }
            }
            eventBus.invokeSubscriber(pendingPost);
            long timeInMethod = SystemClock.uptimeMillis() - started;
            if (timeInMethod >= maxMillisInsideHandleMessage) {
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
                rescheduled = true;
                return;
            }
        }
    } finally {
        handlerActive = rescheduled;
    }
}
```

在 handleMessage 方法内部停留时间不能大于 10 毫秒，从

##### MAIN_ORDERED

新增加的模式，但是当前版本的实现还不完善。



##### ASYN

将事件添加 cachedThreadPool 中执行(如果当前有空闲线程，则复用空闲线程，如果没有就创建新线程)

```java
public void enqueue(Subscription subscription, Object event) {
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
    queue.enqueue(pendingPost);
    eventBus.getExecutorService().execute(this);//放到默认的 cachedThreaPool 中执行
}

@Override
public void run() {
    PendingPost pendingPost = queue.poll();
    if(pendingPost == null) {
        throw new IllegalStateException("No pending post available");
    }
    eventBus.invokeSubscriber(pendingPost);
}
```



#### 3.反射调用订阅方法

org.greenrobot.eventbus.EventBus#invokeSubscriber(org.greenrobot.eventbus.PendingPost)

```java
void invokeSubscriber(PendingPost pendingPost) {
    Object event = pendingPost.event;
    Subscription subscription = pendingPost.subscription;
    PendingPost.releasePendingPost(pendingPost);
    if (subscription.active) {
        invokeSubscriber(subscription, event);
    }
}
```



```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

切换到指定线程之后，通过反射的方法，调用订阅方法。





## 四、总结

EventBus 的实现原理可以归结为以下三点

- 注册—>扫描订阅方法，添加到订阅方法列表中
- 发送事件—>根据事件的类型，遍历方法列表，反射调用订阅方法
- 解注册—>从订阅方法列表中移除相应的订阅方法

`EventBus` 虽然不是标准的观察者模式的实现, 但是它的整体就是一个发布 / 订阅框架, 也拥有观察者模式的优点, 比如: 发布者和订阅者的解耦。



## 五、参考资料与学习资源推荐

- [EventBus 源码解析](http://p.codekk.com/blogs/detail/54cfab086c4761e5001b2538)
- [EventBus 高效使用及源码解析-知乎专栏](https://zhuanlan.zhihu.com/p/22489785)
- [EventBus 3.0 源码分析](https://www.jianshu.com/p/f057c460c77e)
- [EventBus 3.0 源码分析（二）注解处理器的使用](http://xuchongyang.com/2017/07/17/EventBus-3-0-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%8C%EF%BC%89%E7%BC%96%E8%AF%91%E6%97%B6%E6%B3%A8%E8%A7%A3%E5%92%8C%E8%BF%90%E8%A1%8C%E6%97%B6%E6%B3%A8%E8%A7%A3/)



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我。谢谢！