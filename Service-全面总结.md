---
title: Service 全面总结
date: 2017-05-03 23:36:48
tags: 
- Android 基础
categories: 
- Android 基础


---


Service 是一种计算型的组件。

## 一、使用场景

### 线程还是服务？

简单地说，服务是一种**不需要用户交互也可在后台运行的组件**。 因此，我们应该仅在必要时才创建服务。

如需在主线程外部（也就是工作线程）执行工作，并且**只是在用户正在与应用交互时才有此需要，则应创建新线程而非服务**。

如果确实要使用服务，则**默认情况下，它仍会在应用的主线程中运行**，因此，如果服务执行的是密集型或阻塞性操作，那么应该在服务内创建新线程。

> 注意：默认情况下，服务与服务声明所在的应用运行于同一进程，而且运行于该应用的主线程中。
> 为了避免影响应用性能，您应在服务内启动新线程。



<!--more-->

因为默认情况下，它会在应用不管是否用服务，耗时的操作都是需要另外起一个线程的。

1. 从概念上看，服务是一个**组件**，线程是操作系统的最小执行单位。
2. 直接起 Thread 的**可控性**没那么高，它所依附的 Activity finish 掉之后，如果你没有主动停止 Thread 或者 Thread 里的 run 方法没有执行完毕的话，Thread 也会一直执行。你没有办法在不同的 Activity 中对同一 Thread 进行控制。
3. 可以**提高进程的优先级**，使得**进程**没那么容易被 kill。如果是一个长期的任务（比如说长轮询），那么用 Service 去做比较合适。
   - 通过在 manifest 里声明 Service，把需要后台相对长期运行的逻辑放在 Service 里，你便获得了这样的保障：只要系统内存不是极端不够用，你的 Service 一定不会被 kill 掉。对系统而言，当看到一个进程里有 Service 在运行，这个进程就具有较高的优先级，会在内存不足被杀的行列里排得比较靠后。
4. 从交互的角度看，如果是**不需要用户交互也可以在后台运行**，那么可以用服务。如果是在用户与应用交互时（比如说：在设置页面修改用户信息，同步到服务器上面去）那么可以用线程。如果确定要用 Thread，还可以考虑使用 `AsyncTask` 或 `HandlerThread`，而非传统的 `Thread` 类。



所以直接用线程还是起一个服务关键得看任务的类型。是否是长期耗时的任务，是否需要跟用户交互？

## 二、3 种启动方式

### 方式一、start Service

- 调用者**只是负责启动它和停止它**，或者在启动它的时候通过 Intent 传递一点数据给它，除此之外，两者没有数据交换、**没有其他的功能调用**，这两个组件之间基本上互不影响。
  - 如果希望服务返回结果，则启动服务的客户端可以**为广播创建一个 PendingIntent** （使用 getBroadcast()），并==通过启动服务的 Intent 传递给服务==。然后，服务就可以使用广播传递结果。可以获取到 pendingIntent，然后呢？
    - 服务内部也可以直接构造 PendingIntent 啊。
      - 如果是通过客户端的传递的，那么有多个客户端，可以根据传递的 intent 分别处理？
- 如果因为系统内存不足被 kill，之后的具体行为与 onStartCommand 方法的返回值相匹配。

#### 生命周期：

多个服务启动请求会导致多次对服务的 onStartCommand() 进行相应的调用。但是，**要停止服务，只需一个服务停止请求（使用 stopSelf() 或 stopService()）即可**。

### 方式二、bind Service

- 其他组件通过调用 bindService() 绑定 Service，让它运行起来；再通过调用 unbindService()解除绑定。
- 组件和 Service 之间的调用是通过 Binder 来进行的。我们**可以把 Binder 看作是一个连接其他组件和 Service 的桥梁**。
  - 需要注意的是，如果是在同一个进程内，可以**直接继承 Binder**,然后在里面实现提供给调用者的功能。
  - 如果是跨进程，可以使用 aidl。 将要提供给调用方的功能接口写在 aidl 文件中。build 之后，继承 Stub ,实现相应的功能。
    - 调用方在 `onServiceConnected` 方法中，获取到服务端提供的 IBinder 对象就可以像调用普通函数一样，调用相应的方法了。
      - 如果是远程调用的话，将对方提供的 IBinder 类 通过 asInterface 方法进行转换。如果是在服务端，返回 binder 实体，如果是在客户端，返回 binder Proxy。
  - 如果需要提供接口给调用方使用，可以通过 onBind 方法中返回一个可供调用者使用的 Binder 对象。

需要注意的是，**如果用户主动解除绑定，onServiceDisconnected()是不会被触发的**。

如果组件是通过调用 bindService() 来创建服务（且未调用 onStartCommand()，则服务只会在该组件与其绑定时运行。**一旦该服务与所有客户端之间的绑定全部取消，系统便会销毁**。

即使为服务启用了绑定，一旦服务收到对 onStartCommand() 的调用（也就是在 bind 之后又使用 start 的方式去启动应用），则必须手动停止服务。（内部调用 stopSelf 或者 由另一个组件通过调用`stopService()` 来停止它）

**多个客户端先后进行 bindService 只有第一个调用 bindService 时会调用 Service#onBind 方法，后续的都是直接在 ServiceConnection 中返回**。

**绑定是异步的**。bindService() 会立即返回，但是「不会」使 IBinder 返回客户端。要接收 IBinder，客户端必须创建一个 ServiceConnection 实例，并将其传递给 bindService()。ServiceConnection 包括一个回调方法，系统通过调用它来传递 IBinder。
IBinder 返回是异步的。

> 注：只有 Activity、Service 和 ContentProvider 可以绑定到服务 — 无法从广播接收器绑定到服务。



创建支持 bindService 的 Service 的关键在于，自定义一个直接/间接实现了 IBinder 接口的类（编写功能代码提供相应的功能）然后通过 onBind 方法把该类的实例传递给客户端。



注意：Service.onBind如果返回null，则调用 bindService **会启动 Service，但不会连接上 Service**，因此 ServiceConnection.onServiceConnected 不会被调用，但你任然需要使用 unbindService 函数断开它，这样 Service 才会停止。

 



#### 创建提供绑定的 Service

创建提供绑定的服务时，您**必须提供 IBinder**，用以**提供客户端用来与服务进行交互的接口**。 您可以**通过三种方法定义接口**：

##### 1. 直接继承 Binder 类

**适用场景**：
服务是**供单个应用专用，并且在与客户端相同的进程中运行**（常见情况），则应通过扩展 Binder 类并从 onBind() 返回它的一个实例来创建接口。客户端收到 Binder 后，可利用它**直接访问 Binder 实现中乃至 Service 中可用的公共方法**。

如果服务只是 app 的后台工作线程，则优先采用这种方法。 

##### 2. 使用 Messenger

如需**让接口跨不同的进程工作**，则可使用 Messenger 为服务创建接口。服务可以这种方式定义对应于不同类型 Message 对象的 Handler。此 Handler 是 Messenger 的基础，后者随后可与客户端分享一个 IBinder，从而让客户端能利用 Message 对象向服务发送命令。此外，客户端还可定义自有 Messenger，以便服务回传消息。

这是执行进程间通信 (IPC) 的最简单方法，因为 Messenger 会在**单一线程**中创建包含所有请求的队列，这意味着开发者无需进行线程安全控制

##### 3.使用 AIDL

AIDL（Android 接口定义语言）执行所有将对象分解成原语的工作，操作系统可以识别这些原语并将它们编组到各进程中，以执行 IPC。 之前采用 Messenger 的方法实际上是以 AIDL 作为其底层结构。 如上所述，Messenger 会在单一线程中创建包含所有客户端请求的队列，以便服务一次接收一个请求。 不过，如果您想让服务同时处理多个请求，则可直接使用 AIDL。 在此情况下，您的服务必须**具备多线程处理能力**，并采用**线程安全**式设计。

如需直接使用 AIDL，您必须创建一个定义编程接口的 .aidl 文件。Android SDK 工具利用该文件生成一个实现接口并处理 IPC 的抽象类，您随后可在服务内对其进行扩展。

### 方式三、混合型，同时使用 start 和 bind

Service 并不是只能给一个组件使用，它**可以同时服务于多个组件**。
所以一个 Service 既可以是 Start Service，也可以是 Bind Service。只要把两者需要实现的地方都实现了就行。组件 A 可以通过 startService()运行一个 Service，组件 B 可以通过 bindService()再次运行同一个 Service。

### 生命周期

可以绑定到已经使用 startService() 启动的服务。例如，可以通过使用 Intent（标识要播放的音乐）调用 startService() 来启动后台音乐服务。随后，可能在用户需要稍加控制播放器或获取有关当前播放歌曲的信息时，Activity 可以通过调用 bindService() 绑定到服务。在这种情况下，除非所有客户端均取消绑定，否则 stopService() 或 stopSelf() 不会实际停止服务。

1. 先 startService 然后 bindSerive
2. 先 bindService 然后 startService 

在混合模式下，只有等到调用了 stopSelf 或 stopService 以及 所有的客户端都取消绑定，服务才会停止

官方流程图
![image](https://user-images.githubusercontent.com/16668676/35767586-324e8d7a-092a-11e8-9e83-088d73808553.png)



onRebind（Intent intent） 方法与 Activity#onNewIntent 方法类似。
该方法是否被调用取决于 onUnbind 方法是否返回 true。当所有的客户端都调用了 unBindService，而没有调用 stopService 的时候，可以避免这种情况。

- 如果是 true，当有新的调用过来的时候，可以在 onReBind 方法中拿到 intent。
- 如果为 false，当有新的调用过来的时候，会走 onBind
  onRebind() 返回空值，但客户端仍在其 onServiceConnected() 回调中接收 IBinder。

如果要同时支持 start 和 bind，需要同时实现两套方法。

- onStartCommand
- onBind 

这两个方法都含有一个 Intent 参数

### 附：

- 应该始终捕获 DeadObjectException 异常，它们是在连接中断时引发的。这是远程方法引发的唯一异常。
- 对象是跨进程计数的引用。
- 通常应该在客户端生命周期的匹配引入 (bring-up) 和退出 (tear-down) 时刻期间配对绑定和取消绑定。 例如：
  如果您只需要在 Activity 可见时与服务交互，则应在 onStart() 期间绑定，在 onStop() 期间取消绑定。
  如果您希望 Activity 在后台停止运行状态下仍可接收响应，则可在 onCreate() 期间绑定，在 onDestroy() 期间取消绑定。请注意，这意味着您的 Activity 在其整个运行过程中（甚至包括后台运行期间）都需要使用服务，因此如果服务位于其他进程内，那么当您提高该进程的权重时，系统终止该进程的可能性会增加。

> 注：通常情况下，切勿在 Activity 的 onResume() 和 onPause() 期间绑定和取消绑定，因为每一次生命周期转换都会发生这些回调，您应该**使发生在这些转换期间的处理保持在最低水平**。此外，如果您的应用内的多个 Activity 绑定到同一服务，并且其中两个 Activity 之间发生了转换，则如果当前 Activity 在下一个 Activity 绑定（恢复期间）之前取消绑定（暂停期间），系统可能会销毁服务并重建服务。 （Activity 文档中介绍了这种有关 Activity 如何协调其生命周期的 Activity 转换。）

### manifest 文件中声明服务

android:name 属性是唯一必需的属性，用于指定服务的类名。

## 三、创建服务

从传统上讲，您可以扩展两个类来创建启动服务：

### Service

所有服务的基类。扩展此类时，**必须创建一个用于执行所有服务工作的新线程**，因为默认情况下，服务将使用应用的主线程，这会降低应用正在运行的所有 Activity 的性能。

### IntentService

Service 的子类，它使用**工作线程逐一处理**（**串行**处理）所有启动请求。如果您不要求服务同时处理多个请求，这是最好的选择。 您只需实现 onHandleIntent() 方法即可，该方法会接收每个启动请求的 Intent，使您能够执行后台工作。

IntentService 执行以下操作：

- 自动创建默认的工作线程，用于在应用的主线程外执行传递给 onStartCommand() 的所有 Intent。
- 创建工作队列，用于将 Intent 逐一传递给 onHandleIntent() 实现，这样您就永远不必担心多线程问题。
- 在处理完**所有启动请求后停止服务，因此您永远不必调用 stopSelf()**。
- 提供 onBind() 的默认实现（返回 null）。
- 提供 onStartCommand() 的默认实现，可将 Intent 依次发送到工作队列和 onHandleIntent() 实现。

### 在前台中显示、移除

要从前台移除服务，请调用 stopForeground()。此方法采用一个布尔值，指示是否也移除状态栏通知。 此方法不会停止服务。 但是，如果在服务正在前台运行时将其停止，则通知也会被移除

[Start Service 和 Bind Service](http://www.anddle.com/?p=173)

## 四、LocalService 还是 RemoteService？

什么时候选择 local service（即不指定额外的进程），什么时候选择 remote service（额外的进程）？

通常我们会把真的**需要长期运行的 Service（例如 IM 之类）放在单独的进程里**，这样 UI 所在的进程在必要的时候仍然可以被系统 kill 掉来腾出内存。

而 local service 通常用来处理一些**需要短期运行但仍然超出 activity 活动周期的任务**，打个比方，发送短信或彩信。这样的任务执行完以后，service 就可以 stop 自己，仍然不妨碍整个 UI 进程被回收掉。

 





## 五、何时会被 kill？

- 当且仅当内存不足，而此时有***其他进程*的 Activty 具有用户焦点**（或者说，在于用户交互）时，android 系统会强制 stop 服务。
- 如果将**服务绑定到具有用户焦点的 Activity**，则它不太可能会终止；
- 如果**将服务声明为在前台运行**，则它**几乎永远不会终止**。
- 如果服务已启动并要长时间运行，则系统会**随着时间的推移降低服务在后台任务列表中的位置**，而服务也将随之变得非常容易被终止；
- 如果服务是通过 startService 启动的，需要通过重写 onStartCommand 方法妥善处理系统对它的重启。     
  - 因为如果系统终止服务，那么一旦资源变得再次可用，系统便会重启服务（不过这还**取决于从 onStartCommand() 返回的值**）。
  - Q:如果仅通过 bindService 启动，服务被 kill 之后，是否重启仍然取决于 onStartCommand 的返回值吗？
    - 不会，onStartCommand 方法都没有被调用，怎么知道它的返回值呢？

## 参考资料与学习资源推荐

- [Start Service 和 Bind Service](http://www.anddle.com/?p=173)
- [Service](https://developer.android.com/guide/components/services.html?hl=zh-cn#Lifecycle)
- [Bind Service](https://developer.android.com/guide/components/bound-services.html?hl=zh-cn)
- [Android 中 Local Service 最本质的作用是什么？](https://www.zhihu.com/question/19591125)
- [Android 中的 Service 全面总结](https://www.cnblogs.com/newcj/archive/2011/05/30/2061370.html)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源。谢谢！