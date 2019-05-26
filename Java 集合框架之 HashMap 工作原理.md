---
title: Java 集合框架之 HashMap 工作原理
date: 2017-09-10 20:40:54
tags: 
- java集合框架
- 原理分析
categories:
- java集合框架
- 原理分析
---



## 概要

概括的说，`HashMap` 是一个**关联数组、哈希表**，它是**线程不安全**的，允许**key 为 null**,**value 为 null**。遍历时**无序**。 
其底层数据结构是**数组**称之为**哈希桶**，每个**桶里面放的是链表**，链表中的**每个节点**，就是哈希表中的**每个元素**。 
在 JDK8 中，当链表长度达到 8，会转化成红黑树，以提升它的查询、插入效率，它实现了`Map<K,V>, Cloneable, Serializable`接口。

因其底层哈希桶的数据结构是数组，所以也会涉及到**扩容**的问题。

当`HashMap`的容量达到`threshold`域值时，就会触发扩容。扩容前后，哈希桶的**长度一定会是 2 的次方**。 
这样在根据 key 的 hash 值寻找对应的哈希桶时，可以**用位运算替代取余操作**，**更加高效**。

而 key 的 hash 值，并不仅仅只是 key 对象的`hashCode()`方法的返回值，还会经过**扰动函数**的扰动，以使 hash 值更加均衡。 
因为`hashCode()`是`int`类型，取值范围是 40 多亿，只要哈希函数映射的比较均匀松散，碰撞几率是很小的。 

<!--more-->

但就算原本的`hashCode()`取得很好，每个 key 的`hashCode()`不同，但是由于`HashMap`的哈希桶的长度远比 hash 取值范围小，默认是 16，所以当对 hash 值以桶的长度取余，以找到存放该 key 的桶的下标时，由于取余是通过与操作完成的，会忽略 hash 值的高位。因此只有`hashCode()`的低位参加运算，发生不同的 hash 值，但是得到的 index 相同的情况的几率会大大增加，这种情况称之为**hash 碰撞。** 即，碰撞率会增大。

**扰动函数**就是为了解决 hash 碰撞的。它会综合 hash 值高位和低位的特征，并存放在低位，因此在与运算时，相当于高低位一起参与了运算，以减少 hash 碰撞的概率。（在 JDK8 之前，扰动函数会扰动四次，JDK8 简化了这个操作）

执行扩容操作时，会 new 一个新的`Node`数组作为哈希桶，然后将原哈希表中的所有数据(`Node`节点)移动到新的哈希桶中，相当于对原哈希表中所有的数据重新做了一个 put 操作。所以性能消耗很大，**可想而知，在哈希表的容量越大时，性能消耗越明显。**

扩容时，如果发生过哈希碰撞，节点数小于 8 个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。 
因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即 low 位， 或者扩容后的下标，即 high 位。 high 位= low 位+原哈希桶容量 
如果追加节点后，链表数量》=8，则转化为红黑树

由迭代器的实现可以看出，遍历 HashMap 时，顺序是按照哈希桶从低到高，链表从前往后，依次遍历的。属于**无序**集合。

## 实现

### 两个重要的参数

>   An instance of `HashMap` has two parameters that affect its performance: *initial capacity* and *load factor*. **The *capacity* is the number of buckets in the hash table**, and the initial capacity is simply the capacity at the time the hash table is created. The *load factor* is a measure of how full the hash table is allowed to get before its capacity is automatically increased. When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is *rehashed* (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.

-   capacity：容量。也就是哈希桶数组的长度。默认初始化容量为 16。
-   loadFactor：加载因子。  扩容的阈值 `threshold = capacity * loadFactor`
    -   加载因子默认为 0.75，这是时间和空间上的折衷点。大于 0.75，能提高空间利用率，但是会导致查找效率降低。

当 hashMap 中元素的总数大于 capacity * loadFactor  时，就会发生扩容（将 buckets 的数目调整为当前的两倍）。

capacity 被控制为 2 的 n 次方（一定是合数），这有点不合常规。常规的做法是将桶数组的长度设置为素数，因为相对而言使用素数发生冲突的概率要比使用合数要小一些。HashTable 默认初始化容量就为 11（素数，Hashtable 扩容后不能保证还是素数）。之所以这样设计是取模和扩容时进行优化（使用位运算**提高效率**），同时也是为了**减少冲突**，HashMap 定位哈希桶索引位置时，也加入了高位参与运算的过程。





### put 函数的实现

1.8 对 HashMap 的底层实现进行了修改（当相同 hash 值的元素大于 8 个时，会将链表转换为 红黑树，以提高查找效率），所以 put 函数看起来相对复杂了点，通过以下流程图能帮助理解。

图片出自[这篇文章](https://tech.meituan.com/img/java-hashmap/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)。

<img width="858" alt="hashmap put" src="https://user-images.githubusercontent.com/16668676/30247192-dbfc4bf8-9640-11e7-9fd6-0aca25db077c.png">

注意：对于 resize 方法的是否需要执行有两次判断：

第一次判断桶数组是否为空，如果为空，则通过 resize 方法执行初始化操作。
第二次判断是在插入成功后，判断实际存在的键值对数量 size 是否超多了最大容量 threshold，如果超过，进行扩容。

put 方法具体实现如下：

```java
/**
 * 将指定的值与该映射中指定的键关联起来。
 * 如果该 key 对应的 value 存在，则替换旧值
 * */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    java.util.HashMap.Node<K,V>[] tab; java.util.HashMap.Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)//桶数组为空或者长度为 0（说明还未初始化）
        n = (tab = resize()).length;//调用扩容方法进行初始化，并获取初始化后桶数组的长度
    if ((p = tab[i = (n - 1) & hash]) == null)//根据 hash 值与 n-1 进行「模运算」获取插入数组的索引的 i。
        tab[i] = newNode(hash, key, value, null);//创建新结点
    else { //发生碰撞
        java.util.HashMap.Node<K,V> e; K k;

        if (p.hash == hash &&        //key 值相同，替换旧值
                ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof java.util.HashMap.TreeNode)//该链为树
            e = ((java.util.HashMap.TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {//该链为链表
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);//转换为红黑树
                    break;
                }
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
      
        //如果本身存在相同的 key，则将旧值返回
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;//结构化修改次数加 1
    if (++size > threshold)//如果当前数量超过阈值，则进行扩容
        resize();
    afterNodeInsertion(evict);//默认为空实现
    return null;//原先 key 不存在，也就是说没有旧值，直接返回 null
}
```



### get 函数的实现

```java
/**
 * 获取给定 key 的 hash 值，获取相应位置 value
 * */
public V get(Object key) {
    java.util.HashMap.Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final java.util.HashMap.Node<K,V> getNode(int hash, Object key) {
    java.util.HashMap.Node<K,V>[] tab; java.util.HashMap.Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && 
                ((k = first.key) == key || (key != null && key.equals(k))))//先检查对应位置的「头结点」，如果匹配直接返回对应的值
            return first;
        if ((e = first.next) != null) {
            if (first instanceof java.util.HashMap.TreeNode)//该链是红黑树，根据 hash 和 key 到 树中查找相应的 value,直接返回
                return ((java.util.HashMap.TreeNode<K,V>)first).getTreeNode(hash, key);
            //该链是链表，到链表中遍历查找
            do {
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

可以看到 get 方法的大致逻辑是这样的：

-   先计算出当前 key 的 hash 值，然后通过 getNode 方法去获取当前对应的结点
-   如果对应的节点为 null，直接返回 null，
-   否则返回节点中的  value。

getNode 方法的实现思路是这样的：

-   通过 hash 跟 当前容量进行 **与运算** 得到数组下标
-   使用指定下标去获取对应的节点（从头结点开始）
    -   如果首个元素就命中，直接返回结点。
    -   存在冲突：
        如果该 hash 值对应的是一棵**红黑树**，则到红黑树中去获取相应的结点。
        如果该 hash 值对应的是一个**链表**，则遍历该链表，直到取得相应的结点或者到达链尾。

### hash 函数的实现

在 HashMap 中，哈希桶数组 table 的长度 length 大小必须为 2 的 n 次方(一定是合数)，这是一种**非常规的设计**，**常规的设计是把桶的大小设计为素数**。相对来说素数导致冲突的概率要小于合数，具体证明可以参考[这篇文章]( http://blog.csdn.net/liuqiyao_01/article/details/14475159)，Hashtable 初始化桶大小为 11，就是桶大小设计为素数的应用（Hashtable 扩容后不能保证还是素数）。HashMap 采用这种非常规设计，**主要是为了在取模和扩容时做优化，同时为了减少冲突**，HashMap 定位哈希桶索引位置时，也加入了高位参与运算的过程。

通过 `h & (table.length -1)` 来得到该对象的保存位，而 `HashMap` 底层数组的长度总是 2 的 n 次方，这是 HashMap 在速度上的优化。当 length 总是 2 的 n 次方时，h& (length-1) 运算等价于对 length 取模，也就是 h%length，但是&比%具有更高的效率。

-   `h & (length - 1) 《==》 h % length`

在 JDK1.8 的实现中，优化了高位运算的算法，通过 `hashCode()`的高 16 位（其实是完整的） 异或 低 16 位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组 table 的 length 比较小的时候，也能保证考虑到高低 Bit 都参与到 Hash 的计算中，同时不会有太大的开销。

hash 是 int 类型的（32 位），原 hashcode 异或 右移 16 位后的 hashcode，高低位兼顾

-   右移 16 位，那么高 16 位均为 0，所以高十六位的取值由 key 的 hashCode 的高十六位确定，
-   低 16 位的值由 hashcode 的低十六位与高十六位



在 get 和 put 的过程中，计算下标时，先对 hashCode 进行 hash 操作，然后再通过 hash 值进一步计算下标，如下图所示：
![hash](https://cloud.githubusercontent.com/assets/1736354/6957712/293b52fc-d932-11e4-854d-cb47be67949a.png)

在对 hashCode()计算 hash 时具体实现是这样的：

```java
static final int hash(Object key) {
    int h;
  	//因为 h >>> 16 右移了 16 位(高 16 位都为 0)，因此结果的高 16 位仍然是 key.hashCode() 的高 16 位
  	//而低 16 位取决于 key.hashCode() 的高十六位和低十六位进行异或的结果
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

可以看到这个函数大概的作用就是：**高 16bit 不变，低 16bit 和高 16bit 做了一个异或**。这样做的目的就在于你求余的时候**包含了高 16 位和第 16 位的特性** 也就是说你所计算出来的 hash 值包含从而使得你的 hash 值更加「随机」以降低碰撞的概率







#### 计算下标

存储结点时，计算得到的 hash 值可能远大于哈希桶数组的长度，为了避免数组越界，我们需要进行**取模运算**。计算下标的时候，是这样实现的(使用`&`位操作，而非`%`求余)：

```java
(n - 1) & hash 
```

上面的计算实际上等价于`hash % n`，但是前者的效率比较高。前面我们提到过，HashMap 的数组长度一定是 2 的 ？次方。也就是说 (n - 1) 可以化为 0...0011...1，这样跟 hash 进行与运算，就相当于取模运算。

#### Java 8 所做的优化

hash 函数设计得再好，也无法避免冲突的。如何解决冲突也是一门学问。

在 Java 8 之前的实现中是用链表解决冲突的，在产生碰撞的情况下，进行 get 时，两步的时间复杂度是 O(1)+O(n)。因此，当碰撞很厉害的时候 n 很大，O(n)的速度显然是影响速度的。

因此在 Java 8 中，当链表长度大于 8 时，就利用红黑树替换链表，这样复杂度就变成了 O(1)+O(logn)了，这样在 n 很大的时候，能够比较理想的解决这个问题，在[Java 8：HashMap 的性能提升](http://www.importnew.com/14417.html)一文中有性能测试的结果。



### resize （扩容）实现

```java
/**
 * 初始化或者扩容一倍
 * */
final java.util.HashMap.Node<K,V>[] resize() {
    java.util.HashMap.Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;//旧长度
    int oldThr = threshold;//旧阈值
    int newCap, newThr = 0;
    //这一段代码用于确定新容量,并根据新容量重新确定阈值
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {//旧容量大于最大允许容量，将阈值赋值为最大允许容量
            threshold = Integer.MAX_VALUE;
            return oldTab;//直接返回旧表
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)//扩容后小于允许的最大容量，且大于默认初始容量（16），则修改 threshold
            newThr = oldThr << 1; // threshold 增大一倍
    }
    else if (oldThr > 0) // 原有阈值大于 0，但是原容量小于 0
        newCap = oldThr;//将新容量赋值为就旧阈值
    else {               // 原阈值、原容量均为 0 ，使用默认的初始化参数来创建 map
        newCap = DEFAULT_INITIAL_CAPACITY;//默认的初始化容量 （16）
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//新的阈值
    }

    if (newThr == 0) {//如果新阈值为 0
        float ft = (float)newCap * loadFactor;//重新计算
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                (int)ft : Integer.MAX_VALUE); //如果阈值大于最大容量，修改阈值为 int 的最大值(2^31-1)，这样以后就不会扩容了
    }
    
    
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    java.util.HashMap.Node<K,V>[] newTab = (java.util.HashMap.Node<K,V>[])new java.util.HashMap.Node[newCap];//按照新容量 创建数组
    table = newTab;
  
    if (oldTab != null) {//原有的 map 上的元素不为空，将原有 map 上面的数据复制到新的 map 上面
        //遍历旧元素
        for (int j = 0; j < oldCap; ++j) {
            java.util.HashMap.Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;//释放，防止内存泄漏
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;//该桶中只有一个元素，获取的 hash 值并与（新容量 -1）进行 与运算
                else if (e instanceof java.util.HashMap.TreeNode)//如果该元素是一个树节点（说明该桶对应有一棵红黑树），将树存储到新数组上
                    ((java.util.HashMap.TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // 保持顺序
                    //因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即 low 位， 或者扩容后的下标，即 high 位。 high 位 = low 位+原哈希桶容量
                    //低位链表的头结点、尾节点
                    java.util.HashMap.Node<K,V> loHead = null, loTail = null;
                    //高位链表的头结点、尾节点
                    java.util.HashMap.Node<K,V> hiHead = null, hiTail = null;
                    //临时结点
                    java.util.HashMap.Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {//根据 hash 值与 oldCap 的运算结果，将链表中集结的元素分开，可认为结果是随机的
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //在原索引处存放 「低位链表」
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //在原索引加上原容量处，存放「高位链表」
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

我们使用的是 2 次幂的扩展(指长度扩为原来 2 倍)，所以，**元素的位置要么是在原位置，要么是在原位置再移动 2 次幂的位置**。看下图可以明白这句话的意思，n 为 table 的长度，图（a）表示扩容前的 key1 和 key2 两种 key 确定索引位置的示例，图（b）表示扩容后 key1 和 key2 两种 key 确定索引位置的示例，其中 hash1 是 key1 对应的哈希与高位运算结果。

![hashMap 1.8 哈希算法例图 1](https://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE1.png)

元素在重新计算 hash 之后，因为 n 变为 2 倍，那么 n-1 的 mask 范围在高位多 1bit(红色)，因此新的 index 就会发生这样的变化：

![hashMap 1.8 哈希算法例图 2](https://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE2.png)

因此，我们在扩充 HashMap 的时候，不需要像 JDK1.7 的实现那样重新计算 hash，只需要看看原来的 hash 值新增的那个 bit 是 1 还是 0 就好了，**是 0 的话索引没变，是 1 的话索引变成“原索引+oldCap**，可以看看下图为 16 扩充为 32 的 resize 示意图：

![jdk1.8 hashMap 扩容例图](https://tech.meituan.com/img/java-hashmap/jdk1.8%20hashMap%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)

这个设计确实非常的巧妙，既省去了重新计算 hash 值的时间，而且同时，由于新增的 1bit 是 0 还是 1 可以认为是随机的，因此 resize 的过程，均匀的把之前的冲突的节点分散到新的 bucket 了。这一块就是 JDK1.8 新增的优化点。



### 树化与链表化

当冲突链表长度达到 8 的时候，会调用 `treeifyBin` 尝试将链表转换为树。

如果当前数组的长度小于 64 的话，只会触发扩容，而不会将链表转换为红黑树。



```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)//在「树化」之前先判断，当前的容量
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```



当移除元素或者是扩容的时候，如果树的大小<=6，会调用 HashMap.TreeNode#untreeify 方法将红黑树转换为链表。



#### 为什么要进行「树化」呢？

一方面是为了保证性能，如果同一个桶上面的冲突都通过链表连接，链表的查询是线性的，同一个桶上冲突较多的时候，会严重影响存取的性能。

另一个原因其实也是由性能问题引发的哈希碰撞拒绝服务攻击，因为构造哈希冲突的数据并不是困难的事情，恶意代码可以构造这样的数据大量与服务端交互，导致服务端CPU 大量占用。

## 常见问题

此部分内容参考自[HashMap 的工作原理](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

**1. 什么时候会使用 HashMap？他有什么特点？**
是基于 Map 接口的实现，存储键值对时，它可以接收 null 的键值，是非同步的，HashMap 存储着 Entry(hash, key, value, next)对象。

**2. 你知道 HashMap 的工作原理吗？**
通过 hash 的方法，通过 put 和 get 存储和获取对象。存储对象时，我们将 K/V 传给 put 方法时，它调用 hashCode 计算 hash 从而得到 bucket 位置，进一步存储，HashMap 会根据当前 bucket 的占用情况自动调整容量(超过 Load Facotr 则 resize 为原来的 2 倍)。获取对象时，我们将 K 传给 get，它调用 hashCode 计算 hash 从而得到 bucket 位置，并进一步调用 equals()方法确定键值对。如果发生碰撞的时候，Hashmap 通过链表将产生碰撞冲突的元素组织起来，在 Java 8 中，如果一个 bucket 中碰撞冲突的元素超过某个限制(默认是 8)，则使用红黑树来替换链表，从而提高速度。

**3. 你知道 get 和 put 的原理吗？equals()和 hashCode()的都有什么作用？**
通过对 key 的 hashCode()进行 hashing，并计算下标( n-1 & hash)，从而获得 buckets 的位置。如果产生碰撞，则利用 key.equals()方法去链表或树中去查找对应的节点

**4. 你知道 hash 的实现吗？为什么要这样实现？**
在 Java 1.8 的实现中，是通过 hashCode()的高 16 位异或低 16 位实现的：`(h = k.hashCode()) ^ (h >>> 16)`，主要是从速度、功效、质量来考虑的，这么做可以在 bucket 的 n 比较小的时候，也能保证考虑到高低 bit 都参与到 hash 的计算中，同时不会有太大的开销。

**5. 如果 HashMap 的大小超过了负载因子(load factor)定义的容量，怎么办？**
如果超过了负载因子(默认 0.75)，则会重新 resize 一个原来长度两倍的 HashMap，并且重新调用 hash 方法。

## 参考资料与学习资源推荐

-   [面试必备：HashMap 源码解析（JDK8）](http://blog.csdn.net/zxt0601/article/details/77413921)
-   [HashMap 的 hash 函数原理](https://www.zhihu.com/question/20733617)
-   [重新认识 HashMap](https://tech.meituan.com/java-hashmap.html)
-   [HashMap 的工作原理](http://yikun.github.io/2015/04/01/Java-HashMap 工作原理及实现/)



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，谢谢！

