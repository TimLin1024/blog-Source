---
title: IPC 机制
date: 2017-05-04 00:14:59
tags: 
- Android 进阶
- IPC
categories: 
- Android 进阶
- IPC
---



## Android IPC简介 
IPC 是 Inter-Process Communication 的缩写，含义为 进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。
a
IPC 不是 Android 所独有的，任何一个操作系统都需要有相应的 IPC 机制。

在 Android 中最有特色的进程间通信方式就是 Binder 了， **通过 Binder 可以轻松实现进程间通信**。

除了 Binder ，Android 还支持 Socket， 通过 Socket 也可以实现任意**两个终端之间**的通信，当然一个设备的两个进程之间的也可以通过 Socket 进行通信。

多进程的情况分两种：
1. 一个应用因为某些原因自身需要采用多进程模式来实现
    - 可能的原因如下：
        - 有些模块需要运行在单独的进程中
        - 加大一个应用可使用的内存
2. 当前应用需要向其他应用获取数据。


<!--more-->

## 2 Android中的多进程模式 
通过给四大组件指定 `android:process `属性，可以轻易地开启多线程模式。

### 1 开启多进程模式 
一般地，在 Android 中**多进程**是指一个应用中存在多个进程的情况（此处不讨论两个应用之间的多进程情况）

首先，在 Android 中使用多进程只有一种方法， 在 manifest 文件中，给四大组件 指定 `android:process` 属性。
- 其实还有一种非常规的多进程方法—— 通过 JNI 在 native 层去 fork 一个新的进程。

根据 process 的属性值不同，创建不同的进程。

#### 问题：使用 「：xxx」与 使用 「xxx.xxx.xxx」两种方式有区别吗？

1. 「：」的含义是指在当前的进程名前面附上当前的包名，即 这是一种简写的方法
2. 进程名以「：」开头的进程属于**当前应用的私有进程**，其他应用的组件不可以和它跑在同一个进程中，
    - 而进程名不以「：」开头的进程属于**全局进程**，其他应用通过 ShareUID 方式可以和它跑在同一个进程中。



Android 系统会为每个应用分配一个 唯一的 UID，具有相同 UID 的应用才能共享数据。

- 两个应用跑在同一个进程中 需要 
    - 这两个应用有 **相同的 shareUID**
    - 并且 **签名相同**。
- 这种情况下，可以互相访问对方的私有目录，如 data 目录。

### 2 多进程模式的运行机制 
==一个进程 = 一个 虚拟机==

Android 为每个应用分配了一个独立的虚拟机，或者说为每个进程分配一个独立的虚拟机，  
不同虚拟机在内存分配上有不同的地址空间，这就导致了不同虚拟机范文同一个类的对象会产生多份副本。

一般来说，使用**多进程**会造成以下几方面问题
1. 静态成员和单例模式完全失效
2. 线程同步机制完全失效
3. SharePreferences 的可靠性下降，好像有多进程模式？有，但是不稳定，所以系统已经不推荐使用了。
4. Application 会多次创建

可以这样理解同一个应用间的多进程：
- 它就相当于两个不同的应用采用了 ShareUID 的模式，这样能够更加直接地理解**多进程模式的本质**。


## 3 IPC基础概念介绍 
### 3.1 Serializable接口 
**Serializable接口** ：Java 所提供的一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。

#### 使用：
    - 只要在类的声明中指明一个类似下面的标识即可自动实现 *默认的* 序列化过程（可以自定义）。
    - `private static final long serialVersionUID = 87138828109787189L;`

这个 `serialVersionUID` 可以没有，但是这样会对反序列化过程产生影响。
- `serialVersionUID` 是用来辅助序列化过程的，原则上序列化后的数据中的 `serialVersionUID` 只有和当前类的 `serialVersionUID` 相同才能正常地被反序列化。

##### **`serialVersionUID` 的详细工作机制**：
- 序列化是把当前类的 `serialVersionUID` 写入序列化的文件中（也可能是其他中介）当反序列化时会去检测文件中的`serialVersionUID`，看它是否和当前类的`serialVersionUID`一致，如果一致就说明序列化的版本与当前类的版本是相同的，这时可以成功反序列化。
- 否则说明当前类和序列化的类相比发生了 某些变化，例如：成员变量的数量、类型可能发生了改变，这时无法正常序列化，会报如下错误：
```
Exception in thread "main" java.io.InvalidClassException: projectname.clasname; local class incompatible: stream classdesc serialVersionUID = -6009442170907349114, local class serialVersionUID = 6529685098267757690
```
一般而言，我们应该手动指定 `serialVersionUID` 的值，比如 1L，  
也可以让 Eclipse **根据当前类的的结构**去自动生成它的 hash 值，这样序列化和反序列化时两者的 `serialVersionUID`是相同的，可以正常进行反序列化。

如果类的结构发生了改变，比如增加/删除了 某些成员变量，那么系统就会重新计算当前类的 hash 值并把它赋值给 `serialVersionUID` ，这个时候当前类的 `serialVersionUID` 就会和序列化数据不一致，于是反序列化失败。
- 如果手动指定了`serialVersionUID` 就不会有这个问题。

但是如果类结构发生了 ==非常规性改变==，比如修改了类名、修改了成员变量类型，这个时候尽管 `serialVersionUID` 验证通过了，但是反序列化过程还是会失败，因为类结构发生了毁灭性变化，根本无法从老版本的数据中还原出一个新的类结构的对象。

综上，类结构不变且类的版本不变（没有增加或者删除成员变量）的情况下，不指定 serialVersionUID 是没有问题的，但是为了提高稳定性，还是要指定。


**两点要注意的**：
1. 静态成员变量属于类不属于对象，所以**不会参与**序列化过程。
2. 用 transient 关键字标记的 成员变量不参与序列化过程。

系统的默认序列化过程也是可以改变的。只要重新实现 `writeObject` 和 `readObject` 方法即可。

##### 如何进行对象的序列化和反序列化？  
采用 `ObjectOutputStream`  `ObjectInputStream ` 即可轻松实现。
```
User user = new User(name);
ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("cache.txt"));
objectOutputStream.writeObject(user);
objectOutputStream.close();

ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("cache.txt"));
User newUser = objectInputStream.readObject();
objectInputStream.close();
```


### 3.2 Parcelable接口
通过实现Parcelable接口序列化对象的步骤：   
1、声明实现接口Parcelable   
2、实现Parcelable的方法 **writeToParcel**，将你的对象**序列化**为一个Parcel对象   
3、实例化静态内部对象CREATOR实现接口Parcelable.Creator  ，实现 反序列化

Parcelable 的方法说明

| 方法                                    | 功能                                       | 标记位                           |      |
| ------------------------------------- | ---------------------------------------- | ----------------------------- | ---- |
| createFromParcel                      | 从序列化后的对象中创建原始对象                          |                               |      |
| newArray                              | 创建指定长度的原始对象数组                            |                               |      |
| User(Parcel in)                       | 从序列化后的对象中创建原始对象                          |                               |      |
| writeToParcel(Parcel dest, int flags) | 将当前的对象写入序列化结构中，其中 flags 标识有两种值，0 或者 1； 为 1 时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况都为 0 | PARCELABLE_WRITE_RETURN_VALUE |      |
| describeContents                      | 返回当前对象的描述，仅当当前对象中存在 文件描述符，才返回 1，几乎所有情况都返回 0 | CONTENT_FILE_DESCRIPTOR       |      |


新建一个类，实现 Parcelable 接口，然后写好成员变量，再 `alt + enter` 自动补全代码。
```java
public class User implements Parcelable {

    private String mName;
    private int mAge;
    private String mAddress;
    private Book mBook;

    protected User(Parcel in) {//从序列化后的对象中创建原始对象
        mName = in.readString();
        mAge = in.readInt();
        mAddress = in.readString();
        mBook = in.readParcelable(Book.class.getClassLoader());//因为 book 也是一个可序列化对象，所以它的反序列化过程需要传递当前线程的上下文 类加载器
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {//将当前的对象写入序列化结构中，其中 flags 标识有两种值，0 或者 1； 为 1 时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况都为 0
        dest.writeString(mName);
        dest.writeInt(mAge);
        dest.writeString(mAddress);
        dest.writeParcelable(mBook, flags); 
    }

    @Override
    public int describeContents() {//如无特殊情况，这个方法都是返回 0，仅当当前对象中存在 文件描述符，才返回 1
        return 0;
    }
    //反序列化
    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) { 
            return new User(in);
        }

        @Override
        public User[] newArray(int size) {//创建指定长度的原始对象数组
            return new User[size];
        }
    };
}
```



#### 对比
- Serializable 是 Java 中的序列化接口，使用起来简单但是==开销很大==，序列化和反序列化需要大量的 I/O 操作。
- 而 Parcelable 是 Android 中序列化方式，更适合使用在 Android 平台上，==效率高==。 缺点：使用起来麻烦。
    - Parcelable 主要用在**内存序列化**上
- 通过 Parcelable 将对象序列化到**存储设备**中 或者将对象序列化后存储**通过网络传输**也都是可以的，但是这个过程会稍显复杂，在这两种情况下，建议使用 Serializable 接口。

### 3.3 Binder 
直观来说，Binder 是 Android 的一个类，它实现了 IBinder 接口。
- 从 IPC 角度来说， Binder 是 Android 中的**一种跨进程通信方式**
- Binder 还可以理解为一种**虚拟的物理设备**，它的设备驱动是 /dev/binder， 该通信方式在 Linux 上没有。
- 从 Android FrameWork 角度来看， Binder 是 ServiceManager 连接各种 Manager (ActivityManager、 WindowManager， 等等)和相应的 ManagerService 的**桥梁**
- 从 Android 应用层 来说，Binder 是**客户端和服务端进行通信的媒介**，当 bindService 的时候，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务端提供的数据或服务（包括普通的服务 和 AIDL服务）


Android 开发中，Binder 主要用在 Service 中，包括 AIDL 和 Messenger，
- 普通 Service 中的 Binder 不涉及进程间通信，所以较为简单，无法涉及核心
- Messenger 的底层其实是 AIDL

示例中，编写代码时，查找 aidl 生成的接口，Project ==》app==》build ==》source==》debug==》com.xxx.xx.xx==》 目标类

- ==注意==：AIDL 的特殊之处：尽管两个类都位于同一个包中，还是需要导入相应的类。

==所有==**可以在 Binder 中传输的接口**都需要继承自 IInterface 接口。


栗子
```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: D:\\Android_RDC_project\\AndroidLearning\\app\\src\\main\\aidl\\com\\android\\rdc\\androidlearning\\IBookManager.aidl
 */
package com.android.rdc.androidlearning;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.android.rdc.androidlearning.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.android.rdc.androidlearning.IBookManager";//Binder 的唯一标识，一般用当前 Binder 的全类名表示

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 将（服务端的 Binder 对象）转换为（客户端所需的 AIDL 类型的对象）
         * 若服务端与 客户端处在同一个进程，则返回服务端的 Stub 对象本身
         *                      否则返回系统封装后的 Stub.Proxy
         * Cast an IBinder object into an com.android.rdc.androidlearning.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.android.rdc.androidlearning.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.android.rdc.androidlearning.IBookManager))) {
                return ((com.android.rdc.androidlearning.IBookManager) iin);
            }
            return new com.android.rdc.androidlearning.IBookManager.Stub.Proxy(obj);
        }

        /**
         * 返回当前的 Binder 对象
         * */
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
         * 运行在『服务端』的 Binder 线程池中，客户端发起 ，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理
         *  通过code 可以确定请求的目标方法
         *  （如果目标方法有参数的话） 从 data 中取出目标方法所需的参数，然后执行
         *  当目标方法执行完毕后，（如果有返回值的话）向 reply 中写入返回值
         *  如果 onTransact 返回 false ，那么客户端的请求失败。因此可以利用这个特性来做权限验证。
         * */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.android.rdc.androidlearning.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.android.rdc.androidlearning.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.android.rdc.androidlearning.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.android.rdc.androidlearning.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * 此方法运行在『客户端』
             * */
            @Override
            public java.util.List<com.android.rdc.androidlearning.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.android.rdc.androidlearning.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);//调用 transact 发起远程请求。然后当前线程挂起，
                    _reply.readException();// RPC 过程返回后，当前线程继续执行
                    _result = _reply.createTypedArrayList(com.android.rdc.androidlearning.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            /**
             * 执行过程与 getBookList 一样，
             * */
            @Override
            public void addBook(com.android.rdc.androidlearning.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.util.List<com.android.rdc.androidlearning.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.android.rdc.androidlearning.Book book) throws android.os.RemoteException;
}
```


## 4 Android中的IPC方式 

### 4.1 使用Bundle 
四大组件中的三大组件（Activity、Service、Receiver）都是**支持在 Intent 中传递 Bundle 数据**的，Bundle 实现了 Parcelable 接口，所以它可以方便地在不同的进程间 传输。

因此，当我们在一个进程中启动了另一个进程的 Activity、Service、Receiver，我们就可以在 Bundle 中附加我们需要传输给远程进程的信息并通过 Intent 传递出去。
- 当然我们所传输的数据必须能够被序列化，如：
    - 基本数据类型
    - 实现了 Parcelable 接口的对象。
    - 实现了 Serialiable 接口的对象。
    - 一些 Android 支持的特殊对象。
        - Charserquence
        - StringArrayList
        - IntegerArrayList
        - Size
        - ...

#### 特殊的使用场景：  
进程 A 要进行计算，并将计算结果发送给进程 B，但是该计算结果不支持放入 Bundle 中。

- 解决：在 A 中，通过 Intent 来启动线程 B 的一个 Service 组件（比如 IntentService），让 Service 在后台计算，计算完以后再启动 B 进程中想要启动的目标组件。

**核心思想**：将原本需要在 A 进程的计算任务转移到 B 进程的服务中，这样就避免了进程间通信的问题，而且代价也不大。


### 4.2 使用文件共享 
两个进程通过读/写同一个文件交换数据。
- 比如，进程 A 写入，进程 B 读取。

背景知识：
- Window 上，一个文件如果被加了**排斥锁**将会导致其他线程无法对其进行访问（包括读和写）。
- Android 基于 Linux，其读/写文件可以无限制地进行，甚至两个线程对同一个文件并行写也是允许的。（尽管这可能会出现一些问题）

应用：
- 序列化一个对象到文件系统中的同时，从另一个进程中恢复这个对象。
    - 另一个进程成功地恢复之前存储的对象的**内容**，但是他们**本质还是两个对象**。

可能存在的问题：  
并发读/写，读出的内容可能不是最新的。
- 解决：
    1. 设法避免
    2. 考虑使用线程同步来限制，多个线程的写操作。 

文件共享适合**对数据同步要求不高**的进程之间进行通信。


#### Sharepreference
SharePreference 是个特例 

底层通过 xml 文件的方式来存储键值对，，一般而言，它位于 `/data/data/当前应用的包名/share_prefs` 目录下，

由于系统对 SharePreference 的读写操作有一定的**缓存策略**，即**在内存*有一份 SharePreferce 的缓存**。**在多进程模式下**，系统对它的读写操作就变得不可靠。高并发状态下，SharePreference 很大几率会丢失数据。
- 不建议在进程间通信使用 SharePreference

### 4.3 使用Messenger 
『信使』
通过它可以在不同进程中传递 Message 对象，在 Message 中放入我们需要传递的数据，就可以实现进程间的数据传递了。

Messenger 一次处理一个请求，所以服务端不用考虑线程同步的问题（因为不存在并发执行的情形）。

#### 1. 服务端进程


####  客户端进程

在 Messager 中进行数据传递必须将数据放入 Messenger 中，
- Message 中支持数据类型就是 Messenger 所支持的传输类型
- Message 中所能使用的载体只有 
    - int what;
    - int ar1
    - int arg2
    - Messenger replyTo
    - Object obj
        - 2 之前 obj 字段不支持跨进程传输
        - 2 之后 obj 也仅**系统提供的**实现 Parcelable 接口的的对象
            - 即 FrameWork class
    - Bundle 
        - 还好我们有 Bundle ，支持比较多的数据类型

如果要让『服务端』能够回复信息。
- 那么当客户端发送消息的时候，需要把 接收服务端回复的 Messenger 通过 Message 的 replyTo 参数传递给服务端。
### 4.4 使用AIDL 

Android 接口定义语言(Android Interface  Definition Language)

Messenger 只能传递消息，不能跨进程调用方法。而且只能串行处理客户端发来的请求。
- 虽然它底层也是使用 AIDL 实现的。


可以使用 AIDL 来实现跨进程的方法调用。

aidl 文件的作用是 sdk 根据它来生成相应的 java 代码，也就是在编译时有用，在编译完之后，即使删除掉 aidl 文件，也是可以的，但是这样的话如果重新编译就没法生成 java 代码了。

将编写的 `.aidl` 文件保存在项目的 `src/` 目录内，当你开发应用时，**SDK 工具**会在项目的 `gen/` 目录中生成 `IBinder` 接口文件。生成的文件名与 `.aidl` 文件名一致，只是使用了 `.java` 扩展名（例如，`IRemoteService.aidl` 生成的文件名是 `IRemoteService.java`）。

如果**使用 Android Studio**，**增量编译几乎会立即生成 Binder 类**。 如果不是使用 Android Studio，则 Gradle 工具会在下一次开发应用时生成 Binder 类 

通常应该在编写完 `.aidl` 文件后立即用 `gradle assembleDebug` （或 `gradle assembleRelease`）编译项目，以便您的代码能够链接到生成的类。



#### 1. 服务端
用一个 Service 来监听客户的连接请求，创建一个 AIDL 文件，将暴露给客户端的接口在这个 AIDL 文件中声明，最后在 Service 实现该接口。
####  2. 客户端
- 绑定服务端 Service，
- 绑定成功后将服务端返回的 IBinder 转化为 AIDL 接口所属的类型
    - 注意不是平时那样强转，而是
        - `IBookManager.Stub.asInterface(iBinder);`
- 接着就可以使用 AIDL 中的方法了。

AIDL 支持的数据类型：
- 基本数据类型
- CharSequence 
- List 只支持 ArrayList
- Map 只支持 HashMap
- Parcelable 
- AIDL：AIDL 接口本身也可以在 AIDL 文件中使用
    - AIDL 中无法使用普通的接口

- 注意：
    - 自定义的 Parcelable对象一定要显式地 import 进来。（不管它们是否与当前的 AIDL 文件位于同一个包中）
    - 如果 AIDL 文件中用到了自定义的 Parcelable 对象，必须使用新建一个和它同名的 aidl 文件，并在其中声明它为 parcelable 类型
        - 如 Book.java
            - Book.aidl
                - 要在其中声明 pacelable Book;


基本数据类型以外的参数需要==标上方向==
- in 输入型参数，类似于普通方法的参数
- out 输出型参数， 类似于返回值
- inout 输入输入型参数

所以说，输入输出是针对这个方法而言的，而不是 C/S 结构中的输入/输出。

AIDL 接口中只支持方法，不支持声明静态变量（==跟传统的接口有区别==）
- AIDL 包结构在客户端与服务端要保持一致，否则运行会出错。
    - 因为客户端需要反序列化服务端中和  aidl 接口相关的类。

AIDL 接口中所支持的是抽象的 List ，而 List 只是一个接口，虽然服务端使用 CopyOnWriteArrayList ,但是在 Binder 中会按照 List 的规范去访问数据并最终形成一个 ArrayList 传递给客户端。
- ConcurrentHashMap 同理

在**服务端调用客户端**（注册时的使用的接口中)的方法，在客户端的 Binder 线程池中执行。

对象是不能直接跨进程传输的，**对象的跨进程传输本质上都是反序列化过程**

RemoteCallBackList 是系统专门用来发删除跨进程的 listener 的接口。
- 它是一个泛型，支持管理任意的 AIDL 接口
    - `public class RemoteCallBackList<E extends IInterface>`
    - 当客户端进程终止之后， RemoteCallBackList 能够自动溢出客户端所注册的 listener
    - 其内部有一个 `ArrayMap<IBinder,CallBack>`用来专门保存所有的 AIDL 回调
        - CallBack 封装了真正的远程 listener。当客户端注册 listener 的时候，会将其中的信息存入 mCallBack 中
```
IBinder key = listener.asBinder();
callBack value = new Callback(listener,cookie);
```
多次跨进程传输客户端的同一个对象会在服务端生成不同的对象，但是这些新生成的对象有一个 共同点，那就是==它们底层的 Binder 对象是同一个==


==注意==:
- RemoteCallBackList 使用方式比较特别。并不像 list 那样
    - beginBroadcast()
    - finishBroadcast()
    - 这两个方法==一定要配对使用==
    - getBroadcastItem()
```java
final int N = mCallBackList.beginBroadcast();
for (int i = 0; i < N; i++) {
    IOnNewBookArrivedListener l = mCallBackList.getBroadcastItem(i);
    if (l != null) {
        l.onNewBookArrived(book);
    }
}
mCallBackList.finishBroadcast();
```
客户端的 `onServiceConnected 和  onServiceDisconnected(ComponentName name) `都执行在 UI 线程中，不可以在里面调用服务端耗时的方法。

服务端方法本身运行在服务端的 Binder 线程池中，所以服务端方法本身即可执行大量耗时操作。
- 不要在服务端方法中开线程去执行异步任务。

服务端也有可能运行在 UI 线程，这时要尽量避免调用耗时方法。

Binder 死亡后





进程空间分为用户空间和内核空间
- 用户空间，数据互相隔离
- 内核空间 数据共享

通过内核 实现跨进程调用

进程 A 调用进程 B 的函数
- 知道调用的是哪一个对象的哪一个方法
- 传递方法参数
- 进程 B 执行完之后，返回相应的数据给进程 A。

本质上就是 数据的传递。

客户端调用的进程 Binder 对象的引用（一个代理），实际执行还是在服务端的。

Client 进程的的操作实际上对代理对象的操作，代理对象利用 Binder 驱动找到真正的 Binder对象，并通知 Server 进程完成操作。

采用的是 C/S 的架构。

![原理图](http://upload-images.jianshu.io/upload_images/944365-62fdab905e7d2706.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![流程图](http://upload-images.jianshu.io/upload_images/944365-2f530e964ffab8d7.png?imageMogr2/auto-orient/strip%7CimageView2/2)





### 4.5 使用ContentProvider 

待补充

### 4.6 使用Socket 

待补充

## 参考资料与学习资源推荐

-   《Android开发艺术探索》

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源。谢谢！

## 