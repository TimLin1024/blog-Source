---
title: Java 集合框架之 ArrayList
date: 2017-09-06 20:59:19
tags: 
- java集合框架
- 原理分析
categories:
- java集合框架
- 原理分析
---

## 概要

ArrayList 是一个动态数组，它是线程不安全的，允许元素为 null。它的底层数据结构是数组，ArrayList 实现了 `List<E>, RandomAccess, Cloneable, java.io.Serializable` 接口，其中 RandomAccess 代表了其拥有随机快速访问的能力，`ArrayList` 可以以 O(1) 的时间复杂度去根据下标访问元素。

### 时间、空间效率

因为数组内存的连续，可以根据下标以 O1 的时间改查元素，因此**时间效率很高**

同样也因为数组要占据一块连续的内存空间，所以它也有数组的缺点——**空间效率不高**。

<!--more-->

### 性能

当集合中的元素超出容量时，会进行**扩容操作**，扩容操作是一个性能消耗较大的地方，所以如果能预知数据的规模，最好在初始化时通过 `public ArrayList(int initialCapacity)` 构造方法指定 ArrayList 的大小，来构造 ArrayList 实例，以**减少扩容次数，提高效率**。

在添加大量元素前，应用程序也可以使用 ensureCapacity 操作来增加 ArrayList 实例的容量，这可以减少递增式再分配的数量。 不过该方法是 ArrayList 中添加的，List 中没有该方法。所以如果声明的类型为 List 的话，需要进行强转。`((ArrayList)list).ensureCapacity(number);`

当每次修改结构时(添加或者删除元素)，都会修改 modCount。

## 成员变量

```java
private static final int DEFAULT_CAPACITY = 10;//默认初始容量

private static final Object[] EMPTY_ELEMENTDATA = {};//空对象数据，用于空对象，如果指定初始容量为 0 就给元素附一个空对象

/**
 * Shared empty array instance used for default sized empty instances. We
 * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
 * first element is added.
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};//共享的空数组对象，使用该对象用以区分 EMPTY_ELEMENTDATA，从而知道第一次添加元素时，应该初始化的数组长度。

transient Object[] elementData; // 之所以不声明为 private 是为了简化内部类访问，
// 所有原本为默认容量的空数组，在第一次添加元素的时候都会被初始化为长度为默认容量的数组

private int size;//包含元素的数量    
```



## 构造方法

ArrayList 提供了三种方式的构造器，可以构造一个默认初始容量为 10 的空列表、构造一个指定初始容量的空列表以及构造一个包含指定 collection 的元素的列表，这些元素按照该 collection 的迭代器返回它们的顺序排列的。

```java
public ArrayList(int initialCapacity) {//按照指定的初始容量初始化
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];//创建数组
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;//如果指定初始容量为 0 就给元素附一个空对象
    } else {
        throw new IllegalArgumentException("Illegal Capacity: " +
                initialCapacity);
    }
}

public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;//初始化一个初始容量为 10 的 ArrayList
}

//构造一个包含指定 collection 的元素的列表，这些元素按照该 collection 的迭代器返回它们的顺序排列的。
public ArrayList(Collection<? extends E> c) {//从给定的容器中构建一个 ArrayList
    elementData = c.toArray();//将容器对象中的元素转换为数组
    if ((size = elementData.length) != 0) {//长度不为 0，进行判断操作
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);//如果返回的数组不是 Object[].class 类型的，则进行将数组元素复制到类型为 Object 数组上
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;//长度为 0，则将空数组对象赋值给元素数组
    }
}
```

这里要注意的是第三个构造方法中对数组元素类型的判断

```java
 if (elementData.getClass() != Object[].class){
     elementData = Arrays.copyOf(elementData, size, Object[].class);
 } else{
   //...
 }
```

 虽然表面上看起来，c.toArray() 会返回一个  Object[]  对象数组，但是它指向的实际类型并不一定是 Object[]，这样当我们调用 objList[i] =  new Object(); 就会报错   。 比如说如果我们有 1 个 List<String> stringList 对象，当我们调用` Object[] objectArray = stringList.toArray()`的时候，  objectArray 只能存放 String 类型的数据而不能存储其他类型的对象。



## 增

 ArrayList 提供了 add(E e)、add(int index, E element)、addAll(Collection<? extends E> c)、addAll(int index, Collection<? extends E> c)这些添加元素的方法。

```java
/**
 * 添加元素到列表尾部。
 * 先确认 ArrayList 的容量
 * 每次 add 之前，都会判断 add 后的容量，是否需要扩容。
 * */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!! 确保容量足以容纳原有元素加上新增的元素
    elementData[size++] = e;
    return true;
}

/**
 * 添加元素到列表指定位置。需要将该位置右端的所有元素都往右移动一个单位
 * 先确认 ArrayList 的容量
 * */
public void add(int index, E element) {
    rangeCheckForAdd(index);//上下界判断
    ensureCapacityInternal(size + 1);  // Increments modCount!! 判断 add 之后的容量，根据情况进行扩容
    System.arraycopy(elementData, index, elementData, index + 1,// 将 index ~ size-1 范围内的元素复制到 index+1 ~ size 范围中。也就是将 index 及其以后的元素后移一个位置
            size - index);
    elementData[index] = element;//将给定元素添加到指定元素中
    size++;//元素数量 + 1
}
```



```java
/**
 * 添加给定集合中的所有元素到 ArrayList 中
 * */
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();//先转换为数组
    int numNew = a.length;//要添加的元素数量
    ensureCapacityInternal(size + numNew);  // Increments modCount，确保容量足够容纳新添加的所有元素
    System.arraycopy(a, 0, elementData, size, numNew);//将元素添加到列表尾部
    size += numNew;
    return numNew != 0;
}

/**
 * 从指定的位置开始，将指定 collection 中的所有元素插入到此列表中。  
 * */
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);//检查上下界

    Object[] a = c.toArray();//转换为数组
    int numNew = a.length;//增加的数量
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;//要移动的元素数
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                numMoved);//移动数组中需要移动的元素

    System.arraycopy(a, 0, elementData, index, numNew);//插入元素
    size += numNew;
    return numNew != 0;
}
```

## 删

ArrayList 提供了**根据下标**或者**指定对象**两种方式的删除功能。

```java
/**
 * 移除指定位置的元素
 * */
public E remove(int index) {
    rangeCheck(index);//上界判断

    modCount++;//结构修改次数 + 1
    E oldValue = elementData(index);

    int numMoved = size - index - 1;//移动的元素总数
    if (numMoved > 0)
        System.arraycopy(elementData, index + 1, elementData, index,//
                numMoved);//将要删除的元素的后面的所有元素往前移一个单位
    elementData[--size] = null; // clear to let GC do its work 将末元素置为 null，以免内存泄漏

    return oldValue;//返回删除掉的值
}
```



```java
/**
 * 移除此列表中首次出现的指定元素（如果存在的话）。
 * 先找到指定元素在数组中的位置，然后再调用 fastRemove 删除
 * */
public boolean remove(Object o) {
  	// 由于 ArrayList 中允许存放 null，因此下面通过两种情况来分别处理。  
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```



```java
/**
 *  删除 [fromIndex，toIndex) 范围中的所有元素 
 * */
protected void removeRange(int fromIndex, int toIndex) {
    modCount++;
    int numMoved = size - toIndex;
    System.arraycopy(elementData, toIndex, elementData, fromIndex,
            numMoved);

    // clear to let GC do its work
    int newSize = size - (toIndex - fromIndex);
    for (int i = newSize; i < size; i++) {
        elementData[i] = null;
    }
    size = newSize;
}
```

```java
/**
 * 移除 ArrayList 中给定指定集合中的所有元素。
 * */
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
```



```java
/**
 * 批量删除
 * complement 为 false 时，删除指定集合 c 中所有的元素。
 * complement 为 true 时，删除指定集合 c 中以外的所有的元素。
 * */
private boolean batchRemove(Collection<?> c, boolean complement) {// complement 为 true 时为补集
    final Object[] elementData = this.elementData;//将数组引用赋给 elementdata,节省空间
    int r = 0, w = 0;
    boolean modified = false;
    try {
        //遍历，把要保存的元素存在前面
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)//存储要保留的元素。集合 c 中是否包括该元素 == 删除补集？
                elementData[w++] = elementData[r];

    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.

        if (r != size) {// 此段代码作用为：当 c.contains 抛出异常时（此时 r < size ），保持与 AbstractCollection 的行为兼容性
            System.arraycopy(elementData, r,
                    elementData, w,
                    size - r);//将后面的元素移动到 w 后面
            w += size - r;//加上移动的元素数，获取「最后的元素」的下标
        }
        if (w != size) {//清除引用，防止内存泄漏
            // clear to let GC do its work
            for (int i = w; i < size; i++)//将 w 之后的元素都置为 null
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```





### 性能

注意：从数组中移除元素的操作，也会**导致被移除的元素以后的所有元素的向左移动一个位置**。



## 查/获取

 获取此列表中指定位置上的元素。  

```java
public E get(int index) {
    rangeCheck(index);
    checkForComodification();
    return java.util.ArrayList.this.elementData(offset + index);
}
```



### 小结、对比

## 改

```java
public E set(int index, E e) {
    rangeCheck(index);
    checkForComodification();
    E oldValue = java.util.ArrayList.this.elementData(offset + index);
    java.util.ArrayList.this.elementData[offset + index] = e;
    return oldValue;
}
```

### 对比



## 调整数组容量

### 扩容

```java
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}

/**
 * 内部保证容量，如果 ArrayList 是通过无参构造函数创建的，
 * 那么第一次 add 元素的时候就会调用该方法扩容
 * */
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

/**
 * 确保分配指定的容量
 * */
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    //如果指明的最小容量超过数组的长度，就增大容量
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```



每次数组容量的增长大约是其原容量的 1.5 倍。这种操作的代价是很高的，因此在实际使用时，我们应该尽量避免数组容量的扩张。当我们可预知要保存的元素的多少时，要在构造 ArrayList 实例时，就指定其容量，以避免数组扩容的发生。或者根据实际需求，通过调用 ensureCapacity 方法来手动增加 ArrayList 实例的容量。

### 「压缩」

   ArrayList 还给我们提供了将底层数组的容量调整为当前列表保存的实际元素的大小的功能。它可以通过 trimToSize 方法来实现。代码如下：

```java
//将 ArrayList 的容量压缩到当前元素数量，这样可以最大限度节省空间
public void trimToSize() {
    modCount++;//定义与父类 AbstractList 中，记录发生结构化修改的次数。如果不打算提供「快速失败的迭代器」，可忽略此域
 
    if (size < elementData.length) {
        elementData = (size == 0)
                ? EMPTY_ELEMENTDATA
                : Arrays.copyOf(elementData, size);
    }
}
```



## Fail-Fast 机制

ArrayList 也采用了快速失败的机制，通过记录 modCount 参数来实现。在面对并发的修改时，迭代器很快就会完全失败，而不是冒着在将来某个不确定时间发生任意不确定行为的风险。具体介绍请参考我之前的文章[深入 Java 集合学习系列：HashMap 的实现原理](http://zhangshixi.iteye.com/blog/672697) 中的 Fail-Fast 机制。

-   快速失败：当你在迭代一个集合的时候，如果有另一个线程正在修改你正在访问的那个集合时，就会抛出一个 ConcurrentModification 异常。   
    -   在 java.util 包下的都是快速失败。
-   安全失败：你在迭代的时候会去底层集合做一个拷贝，所以你在修改上层集合的时候是不会受影响的，不会抛出 ConcurrentModification 异常。
    -   在 java.util.concurrent 包下的全是安全失败的。

即 抛异常是快速失败（util 包下都是快速失败），不抛异常是安全失败。 Java 版本越往后越「安全」，concurrent 包下面全部为**安全失败**

## 总结

可以看到核心操作在于增加和删除元素。

1. 增删改查中， 增导致**扩容**，则**会修改 modCount**，**删一定会修改**。 **改和查一定不会修改 modCount**。
2. 扩容操作会导致数组复制，**批量删除会导致找出两个集合的交集，以及数组复制操作**，因此，增、删都相对低效。 而 改、查都是很高效的操作。
3. 因此，结合特点，在使用中，以 Android 中最常用的展示列表为例，列表滑动时需要展示每一个 Item（element）的数组，**所以 查 操作是最高频的**。相对来说，**增操作只有在列表加载更多时才会用到** ，而且是在列表尾部插入，所以也不需要移动数据的操作。而删操作则更低频。 故选用 ArrayList 作为保存数据的结构。
4. 和`Vector`的区别，Vector`内部也是数组实现的，区别在于`Vector`在 API 上都加了`synchronized`所以它是线程安全的，以及`Vector`扩容时，是翻倍 size，而`ArrayList`是扩容 50%。





## 参考资料与学习资源推荐

-   [面试必备：ArrayList 源码解析（JDK8）](http://blog.csdn.net/zxt0601/article/details/77281231)
-   [深入 Java 集合学习系列：ArrayList 的实现原理](http://zhangshixi.iteye.com/blog/674856)
-   [Java ArrayList 工作原理及实现](http://yikun.github.io/2015/04/04/Java-ArrayList%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

若本文中有不正确的结论、说法，请大家提出，共同探讨，共同进步，谢谢!