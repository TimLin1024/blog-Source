---
title: String StringBuilder与StringBuffer的区别
date: 2017-09-21 14:26:19
tags: 
- java 基础
categories:
- java 基础
---

## 概述

![image](http://ofucm8avi.bkt.clouddn.com/1.png)

CharSequence 就是字符序列，String, StringBuilder 和 StringBuffer 本质上都是通过字符数组实现的。

1.  String 字符串**常量**（不可变）
2.  StringBuffer 字符串变量（线程安全）
3.  StringBuilder 字符串变量（非线程安全）

<!--more-->

## String 类型和 StringBuilder 类型的主要性能区别

1.  String 是**不可变的对象**,因此在每次对 String 类型进行改变的时候其实都等同于生成了一个新的 String 对象，然后将指针指向新的 String 对象，所以**经常改变内容的字符串最好不要用 String** ，因为每次生成对象都会对系统性能产生影响，特别当内存中无引用对象多了以后， JVM 的 GC 就会开始工作，会进一步消耗性能。
2.  StringBuilder 是可变的， 可以对 StringBuilder **对象本身进行操作**，而不是生成新的对象，再改变对象引用。所以在一般情况下我们推荐使用 StringBuilder ，特别是字符串对象经常改变的情况下。
3.  通常 String 对象的字符串拼接其实是被 JVM 解释成了 StringBuilder 对象的拼接，所以这些时候 String 对象的速度并不会比 StringBuilder 对象慢。

特别是以下的字符串对象生成中，String 效率是远要比 StringBuffer 高的：

```java
String S1 = “This is only a” + “ simple” + “ test”;//直接拼接现有字符串
StringBuffer Sb = new StringBuilder(“This is only a”).append(“ simple”).append(“ test”);
```

这其实涉及到了编译器的优化。在编译期，编译器会把**字符串常量**相加的情况直接变为拼接为目标字符串。也就是说编译之后就获得了目标字符串。

```java
String S1 = “This is only a” + “ simple” + “test”; 
 //其实就是：
String S1 = “This is only a simple test”;
```

所以当然不需要太多的时间了。要注意的是，如果你的字符串是来自**另外的 String 对象**的话（比如下面所列举的这个例子），直接将 String 相加，性能上不会有什么提升。

```java
String S2 = “This is only a”;
String S3 = “ simple”;
String S4 = “ test”;
String S1 = S2 + S3 + S4;
```

这时候 JVM 会规规矩矩的按照原来的方式去做——先创建一个 StringBuilder 对象，然后再进行调用它的 append 方法对 字符串进行拼接。

## StringBuffer

StringBuffer 是线程安全的可变字符序列。一个类似于 String 的字符串缓冲区，但不能修改。虽然在任意时间点上它都包含某种特定的字符序列，但通过某些方法调用可以改变该序列的长度和内容。

可将字符串缓冲区安全地用于多个线程。可以在必要时对这些方法进行同步，因此任意特定实例上的所有操作就好像是以串行顺序发生的，该顺序与所涉及的每个线程进行的方法调用顺序一致。

### 主要操作:

append 和 insert 方法，可重载这些方法，以接受任意类型的数据。

append 方法始终将这些字符**添加到缓冲区的末端**。

insert 方法则在指定的点添加字符。

例如:   
z 引用一个当前内容是“start”的字符串缓冲区对象，    
用 z.append("le") 会使字符串缓冲区包含“startle”，  
而 z.insert(4, "le") 将**更改字符串缓冲区**，使之包含“starlet”。

## StringBuilder

StringBuilder 是 Java5 新增的一个非线程安全的可变的字符序列。可以把它理解为 StringBuffer 的非线程安全版本，因为实际应用中很少有需要对字符串进行同步的情况，所以采用 StringBuilder 的性能更加。在不需要同步，优先采用 StringBuilder。

初始长度分为三种情况:

 如果是调用的是无参构造函数，则默认容量为 16

如果指定了一个大于 0 的 capacity，则初始容量为 指定的 capacity

如果使用了参数类型为  CharSequence/String 的构造函数，则初始容量为`CharSequence的长度 + 16`。



### 扩容

每次 append 的时候都会调用 ensureCapacityInternal ，当前长度 + append 内容总长度作为 minimumCapacity，如果  minimumCapacity 超过了 value 数组的长度，就将value 数组复制到长度为 newCapacity 的新数组，

一般情况下 newCapacity 大于  minimumCapacity

```java
/**
 * For positive values of {@code minimumCapacity}, this method
 * behaves like {@code ensureCapacity}, however it is never
 * synchronized.
 * If {@code minimumCapacity} is non positive due to numeric
 * overflow, this method throws {@code OutOfMemoryError}.
 */
private void ensureCapacityInternal(int minimumCapacity) {
    // overflow-conscious code
    if (minimumCapacity - value.length > 0) {
        value = Arrays.copyOf(value,
                newCapacity(minimumCapacity));
    }
}
```



```java
/**
 * The maximum size of array to allocate (unless necessary).
 * Some VMs reserve some header words in an array.
 * Attempts to allocate larger arrays may result in
 * OutOfMemoryError: Requested array size exceeds VM limit
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 * Returns a capacity at least as large as the given minimum capacity.
 * Returns the current capacity increased by the same amount + 2 if
 * that suffices.
 * Will not return a capacity greater than {@code MAX_ARRAY_SIZE}
 * unless the given minimum capacity is greater than that.
 *
 * @param  minCapacity the desired minimum capacity
 * @throws OutOfMemoryError if minCapacity is less than zero or
 *         greater than Integer.MAX_VALUE
 * 返回一个至少跟 minCapacity 一样大的值。如果空间足够的话，当前的容量会增长一倍 +2 。
 * 除非给定的值超过MAX_ARRAY_SIZE。否则返回的容量不会超过 MAX_ARRAY_SIZE。
 */
private int newCapacity(int minCapacity) {
    // overflow-conscious code。防止溢出
    int newCapacity = (value.length << 1) + 2;//当前数组长度 * 2 + 2
    if (newCapacity - minCapacity < 0) {//如果新长度 < minCapacity，将新长度赋值为 minCapacity
        newCapacity = minCapacity;
    }
    return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
       ? hugeCapacity(minCapacity)//检查溢出。如果溢出了，抛出错误。否则置为 MAX_ARRAY_SIZE
        : newCapacity;//长度在正常范围内，直接使用该长度
}

private int hugeCapacity(int minCapacity) {
    if (Integer.MAX_VALUE - minCapacity < 0) { // overflow
        throw new OutOfMemoryError();
    }
    return (minCapacity > MAX_ARRAY_SIZE)
        ? minCapacity : MAX_ARRAY_SIZE;
}
```











##  参考资料与学习资源推荐

- [Class StringBuilder](https://docs.oracle.com/javase/7/docs/api/java/lang/StringBuilder.html)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！