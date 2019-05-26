---
title: 深入理解 ThreadLocal
date: 2017-08-21 10:53:40
tags: 
- 并发
categories:
- 并发
---


## ThreadLocal 是什么？
ThreadLocal 是一个**线程内部**的**数据存储类**，通过它可以在指定的线程中存储数据，数据存储后，只有在指定线程中才能获取到存储的数据，对于其他线程来说则无法获取到数据。

## 为什么要使用 ThreadLocal？

从定义我们知道 ThreadLocal 是一个用于存储本线程内部数据的类。假设没有 ThreadLocal 的话，每个 Thread 中可以输入自己的一个本地变量，但是在整个 Thread 的生命周期中，如果要穿梭很多 class 的很多 method 来使用这个本地变量的话，就要一直一直向下传送这个变量，显然很麻烦。
那么怎么才能在这个 Thread 的生命中，在任何地方都能够方便的访问到这个变量呢，这时候 ThreadLocal 就诞生了。

<!--more-->

ThreadLocal 就是这么个作用，除此之外和通常使用的本地变量没有任何区别。
也就是说，没有 ThreadLocal 也是可以解决问题的，但是会比较麻烦，ThreadLocal 的作用便是简化线程内部数据的使用流程。

## ThreadLocal 的内部实现
既然是线程的本地变量，那自然与线程有着密切的联系。

打开 Thread 的源码可以看到，源码中有一个类型为 `ThreadLocal.ThreadLocalMap` 的变量
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```



###  ThreadLocal#get 流程

我们以 ThreadLocal#get 方法作为分析的源头。这些方法的逻辑都比较简单，因此直接在注释中说明。可参考小结部分的调用流程图。

```java
public T get() {
    Thread t = Thread.currentThread();//获取当前线程
    ThreadLocalMap map = getMap(t);//获取当前线程的 ThreadLocalMap 对象
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);//获取存储该 ThreadLocal 的 Entry
        if (e != null)
            return (T)e.value;//获取目标值并返回
    }
    return setInitialValue();//设置初始值
}
```
ThreadLocal#getMap，该方法返回当前线程的 ThreadLocalMap  对象。

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

Entry 类继承自弱引用，防止内存泄漏。其中存储了 ThreadLocal 以及对应的值

```java
static class Entry extends WeakReference<ThreadLocal> {
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```

ThreadLocal#setInitialValue

```java
private T setInitialValue() {
    T value = initialValue();//获取默认的初始值
    Thread t = Thread.currentThread();//获取当前线程
    ThreadLocalMap map = getMap(t);//获取当前线程的 ThreadLocalMap 对象
    if (map != null)
        map.set(this, value);//当前线程的 ThreadLocalMap 对象不为空，直接设置给目标对象。
    else
        createMap(t, value);//为当前线程创建 ThreadLocalMap 对象。
    return value;
}
```



ThreadLocal#initialValue

```java
//该方法的调用时机：
//1. 通常该方法只会被调用一次，也就是在第一次初始化时
//2. 调用了 ThreadLocal#remove 方法之后（使得 Entry 被回收），再调用 ThreadLocal#get 方法
protected T initialValue() {
    return null;//；默认实现为返回 null
}
```



ThreadLocal#createMap

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);//调用构造方法初始化
}
```



### ThreadLocal#set 流程

```java
public void set(T value) {
    Thread t = Thread.currentThread();//获取当前线程
    ThreadLocalMap map = getMap(t);//获取当前线程的 ThreadLocalMap 对象
    if (map != null)
        map.set(this, value);//当前线程的 ThreadLocalMap 对象 已经存在直接设置值
    else
        createMap(t, value);//为当前线程创建 ThreadLocalMap 
}
```

可以看到 ThreadLocal#set 方法的逻辑与 ThreadLocal#setInitialValue 方法中的逻辑如出一辙，这里不再赘述。





###  ThreadLocalMap   

ThreadLocalMap 是 ThreadLocal 的一个静态内部类。其中以键值对的形式存储数据。可以将它简单理解为一个 HashMap。ThreadLocal#createMap 方法通过调用 ThreadLocalMap 的构造方法为当前线程创建一个 ThreadLocalMap 对象。

#### ThreadLocalMap 的构造方法

```java
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];//创建一个长度为 16 的 Entry 数组
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);//计算 key 的哈希值
    table[i] = new Entry(firstKey, firstValue);//将 Entry 存到指定位置。
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```
`ThreadLocalMap` 的构造方法主要做了下面几件事：
- 首先创建了一个 `Entry` 数组，
    - `Entry` 是 `ThreadLocalMap` 中的一个静态内部类,它以 `ThreadLocal` 为 key，以要存储的值为 value。
- 然后根据 key 计算 Hash 值
- 接着创建一个 Entry 对象存储在数组中
- 最后设置大小和阈值。


#### ThreadLocalMap.Entry

```java
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```

Entry 作为 ThreadLocalMap 的元素，表示的是一个键值对：ThreadLocal 的弱引用为键，将要用 ThreadLocal 存储的对象为值。

### 解决冲突的方式

线性探测法。



```java
    /*ThreadLocals 依赖于附加到每个线程（Thread.threadLocals和inheritableThreadLocals）的线性探测 HashMap。 ThreadLocal对象作为键，通过threadLocalHashCode进行搜索。这是一个自定义的 hash code（仅在ThreadLocalMaps中有用），可以消除常见情况下的冲突，而在不常见的情况下也能表现良好。
*/
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();
/**
 * The difference between successively generated hash codes - turns
 * implicit sequential thread-local IDs into near-optimally spread
 * multiplicative hash values for power-of-two-sized tables.
 */
private static final int HASH_INCREMENT = 0x61c88647;//（1640531527）十进制


/**
 * Returns the next hash code.
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```





java.lang.ThreadLocal.ThreadLocalMap#getEntryAfterMiss

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

java.lang.ThreadLocal.ThreadLocalMap#nextIndex

```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

## ThreadLocal 会造成内存泄漏？

因为作为 key 的 ThreadLocal 是弱引用，所以一发生 GC ，ThreadLocal 就会被回收，这个时候 Map 中存在一个 Key 为 null 的键值对，但是 **value 仍然被线程强引用着**，那么如果用完ThreadLocal后不主动移除（调用 ThreadLocal#remove 方法），就会造成**短期的内存泄漏**。但事实上，ThreadLocal **用完后主动调 remove** 就能规避这个问题。

注：上面说 value 被线程强引用是因为存在这样一条引用链：栈帧中持有一个当前线程的引用，ThreadLocalMap 被当前线程引用着，ThreadLocalMap 中有指向 Entry 的强引用，Entry 有持有 value 的强引用。具体可参考下图。

![image](https://user-images.githubusercontent.com/16668676/31853072-84517b72-b6b5-11e7-84f1-0a51baaa05a2.png)

当前 Thread 结束以后, Current Thread 就不会存在栈中,强引用断开, Current Thread, Map, value将全部被GC回收。



前面提到不调用 remove 方法 ThreadLocal 会造成短期的内存泄漏，下面就来证明下这个说法是否正确。

### 为什么说是「短期的内存泄漏」呢？

以 ThreadLocal#get 为例

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);//调用 ThreadLcoalMap#getEntry 方法
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}
```

```java
private Entry getEntry(ThreadLocal key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)//命中
        return e;
    else
        return getEntryAfterMiss(key, i, e);//不在数组的相应位置上（可能是冲突）
}
```



`ThreadLocal.ThreadLocalMap#getEntryAfterMiss`

```java
private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);//移除「过期的」Entry，也就是移除 key 为 null 的 Entry
        else
            i = nextIndex(i, len);//加 1 「取模」，获得下一个地址
        e = tab[i];
    }
    return null;
}
```

`ThreadLocal.ThreadLocalMap#expungeStaleEntry`

```java
//移除指定下标对应的的「过期的」Entry，
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 删除过期的 slot 上的 entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // 不断尝试，直到遇到 e 为 null为止。
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);//（staleSlot +1）% len
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {//(i+1)%len
        ThreadLocal k = e.get();//待清除的 entry
        if (k == null) {//key 为 null，说明该 entry 已经过期，需要被移除
            e.value = null;//将 value 置为 null，方便 GC
            tab[i] = null;//解除对 entry 的应用
            size--;
        } else {//key 不为 null 的情况。这一步的操作主要是方便查找，提高命中率
            int h = k.threadLocalHashCode & (len - 1);//通过 hash 值计算下标
            if (h != i) {//下标不相等，说明之前setValue 时发生冲突，往后面存储了。
                tab[i] = null;//将
				//与Knuth 6.4算法R 不同，我们必须扫描直到null，因为可能有多个entry已经过期。
                while (tab[h] != null)//从「本应该对应的下标」开始往后探测，直到有空位
                    h = nextIndex(h, len);
                tab[h] = e;//将 e 存储在当前情况下，最接近自己计算得出位置的那个下标。
        }
    }
    return i;
}
```

从 `ThreadLocal#get` 方法（set 方法也有类似的实现）的调用链中可以看到：在没有主动调用 ThreadLocal#remove 的情况下，ThreadLocalMap后续的get/set中也会探测到那些key为 null的entry，然后将其 value 设置为 null 以帮助GC，因此 **value 在 key 被 GC 后可能还会存活一段时间（也就是说会造成短期的内存泄漏），但最终也会被回收**。这个过程和 `java.util.WeakHashMap` 的实现几乎是一样的。不过为了避免潜在的内存泄漏还是要养成一个习惯，使用完 ThreadLocal 中的value 之后要调用 `ThreadLocal#remove` 。



## 小结 

#### get 方法调用的主要流程

![threadlocal get](https://user-images.githubusercontent.com/16668676/30096800-004b9f36-930d-11e7-9c27-c7f17619c3a7.png)

get 方法中会尝试获取当前线程的 ThreadLocalMap

-   如果 ThreadLocalMap 非空，并且以当前 ThreadLocal 对象为 key 去获取到的 Entry 不为空，就返回该 ThreadLocal 对应的值；
-   否则，先获取默认的初始值（默认实现为空，可以自己重写 initialValue 方法来设置需要的值）
    -   然后判断 ThreadLocalMap 是否为空
        -   如果为空创建一个 ThreadLocalMap 同时将初始值设置进去。
        -   如果不为空，直接把初始值存储在其中。

#### set 方法的调用流程



![threadlocal set](https://user-images.githubusercontent.com/16668676/30096799-0047aeee-930d-11e7-8ac5-1927aadeab2d.png)





## 你可能存在的疑问

### 每个 ThreadLocal 只能放一个对象吗？
每个 ThreadLocal 只能放一个对象。要是需要放其他的对象，就再 new 一个新的 ThreadLocal 出来，这个新的 ThreadLocal 将作为 key,需要放的对象作为value，放在 ThreadLocalMap 中。也就是说一个线程可以含有多个 ThreadLocal 类。

当然也可以根据需要在 ThreadLocal 存放一些容器对象，比如 List、Set、Map，一个 ThreadLocal 存放一个容器对象，借助该容器对象也可以实现存储多个对象。 

### 为什么 ThreadLocal  只存储一个对象却要用一个 ThreadLocalMap 来存储值？

实际上每个线程中都有一个 ThreadLocal.ThreadLocalMap，真正存储数据的类是 ThreadLocalMap ，可以将它看作是一个 HashMap，而 ThreadLocal 是一个**维护类**。我们知道，存储的时候，都是以 ThreadLocal 实例作为 key，然后和 value 一起作为键值对存储到 ThreadLocalMap 中。当我们调用不同 ThreadLocal 的 set 方法时，如果 ThreadLocalMap 不为空，那么直接在里面存储键值对就可以了，不需要再创建新的值。也就是说，**同一个线程上的不同 ThreadLocal 对象，存储的值是在同一个 ThreadLocalMap 上**。




### 没有 ThreadLocal 能不能解决问题？
能。

可以自己定义一个静态的 map，将当前 thread 作为 key，将目标值作为 value，put 到 map 中，这也是一般人的想法。


ThreadLocal 的实现刚好相反，它是在每个线程中有一个 map，而将 ThreadLocal 实例作为 key，这样每个 map 中的项数很少，而且当线程销毁时相应的东西也一起销毁了。
因为各线程访问的 map 是各自不同的 map，所以不需要同步，速度会快些；而如果把所有线程要用的对象都放到一个静态 map中的话 多线程并发访问需要进行同步。


所以说 ThreadLocal 只是实现线程私有变量的一种方式。但是综合来看这种方式相比其他实现方式要更好。

## 典型应用

我们通常会用下面的方式为普通线程创建一个 Looper。

```java
class LooperThread extends Thread {
        public Handler mHandler;
        public void run() {
            Looper.prepare();//为当前线程创建一个 Looper
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };
            Looper.loop();//开启消息循环
        }
    }
```



Android 中每个线程中最多只能有一个 Looper，这种限制就是通过 ThreadLocal 来实现的。

```java
public final class Looper {
  	//创建一个 ThreadLocal 对象，其泛型类型为 Looper，在调用 prepare 之前 sThreadLocal.get() 都返回空。
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  	//...
}
```

创建 Looper 时会同时创建一个泛型类型为 Looper 的 ThreadLocal 对象。

Looper#prepare()，通过 prepare 方法可以为当前线程创建一个 Looper。

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {//如果线程已经存在 Looper 了
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));//为当前线程创建一个 Looper 对象，并将它存储在当前线程的 ThreadLocalMap 中
}
```

Looper#myLooper() ，通过该方法可以获取当前线程的 Looper 对象

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```






## 注意
如果一个对象的引用被多个线程持有，那么即使该对象存在 ThreadLocalMap 中也不是线程的本地变量。

首先，ThreadLocal 不是用来解决共享对象的多线程访问问题的，一般情况下，通过 `ThreadLocal.set()` 到线程中的对象是该线程自己使用的对象，其他线程是不需要访问的，也访问不到的。各个线程中访问的是不同的对象。 

也就是说，其他线程能否访问，还要看你的 set 进去的对象引用是否被其他线程持有。 如果两个线程都存入同一个对象引用，那就会有线程共享问题。



## 总结
我们总结 ThreadLocal 具体是怎么一步一步去为每一个线程创建一个~线程私有变量~的：
- 首先，在每个线程 Thread 内部有一个 `ThreadLocal.ThreadLocalMap` 类型的成员变量 threadLocals，这个 threadLocals 就是用来存储~线程私有变量的~，键值（key）为当前 ThreadLocal 变量，值 value 为~线程的私有变量~（即 T 类型的变量）。
- 初始时，在 Thread 里面，threadLocals 为空，当通过 ThreadLocal 变量调用 get() 方法或者 set() 方法，就会对 Thread 类中的 threadLocals 进行初始化，并且以当前 ThreadLocal 变量为 key，以 ThreadLocal 要保存的~线程私有变量~为 value，存到 threadLocals 中。
    - 注意，如果是 先调用 get() 方法而不是 set() 方法的话，会返回 null
- 然后在当前线程里面，如果要使用~该线程私有变量~，就可以通过 get 方法在 threadLocals 里面查找。





## 参考资料与学习资源推荐
- [正确理解 ThreadLocal](http://www.iteye.com/topic/103804)
- [ThreadLocal and synchronized 补充](http://www.iteye.com/topic/82984)
- [ThreadLocal](http://droidyue.com/blog/2016/03/13/learning-threadlocal-in-java/index.html)
- [Java并发编程：深入剖析ThreadLocal](http://www.cnblogs.com/dolphin0520/p/3920407.html)
- [Android关于ThreadLocal的思考和总结](http://www.jianshu.com/p/95291228aff7)
- [深入理解ThreadLocal](http://vence.github.io/2016/05/28/threadlocal-info/)



如果本文中有不正确的结论、说法，请大家指出，共同探讨，共同进步，谢谢!
