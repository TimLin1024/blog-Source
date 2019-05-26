---
title: Java 集合框架之 LinkedHashMap 工作原理
date: 2017-09-17 19:24:14
tags:
- 集合框架
categories:
- 集合框架
---

## 一、不同类型 Map 的适用场景

- 如果需要使用的Map中的**key无序**，选择`HashMap`；


- 如果要求**key有序**，则选择`TreeMap`。 

  - 但是选择TreeMap就会有**性能问题**，因为TreeMap的get操作的时间复杂度是`O(log(n))`的，相比于HashMap的`O(1)`还是差不少的，

- `LinkedHashMap`的出现就是为了平衡这些因素，使得 能够以**O(1)时间复杂度增加查找元素，又能够保证key的有序性** 此外，LinkedHashMap提供了两种key的顺序：

  1. 按照访问顺序排序（access order）。可以使用这种顺序**实现`LRU（Least Recently Used）`缓存**

  - 因为每次访问后的元素会被移到链表尾部，实现 LRU 只需要移除 链首元素 即可
    - 「一次访问」的定义是什么？
      - put，putIfAbsent，get，getOrDefault，compute，computeIfAbsent，computeIfPresent或merge方法会导致访问相应的条目（假设它在调用完成后存在）。如果值被替换，替换方法只会导致条目的访问

  1. 按照插入顺序排序（insertion orde）。注：同一key的多次插入，并不会影响其顺序。

注：WeakHashMap 也可以用于实现缓存，二者的使用场景不同。WeakHashMap无法像 LinkedHashMap 那样自定义淘汰规则，它的元素会在 gc 的发生的时候被清除。



特别地，对 collection-view 的操作不会影响后台映射的迭代顺序。

- collection-view 是否类似于数据库的视图？

## 二、性能

LinkedHashMap 提供了所有可选的Map操作，并允许使用null元素。与HashMap一样，假设散列函数能够正确地在桶之间分散元素，它为基本操作（添加，包含和删除）提供了恒定的性能。性能可能略低于HashMap的性能，这是因为**维护链接列表的成本增加了**，但有一个**例外**：对LinkedHashMap的collection-view的迭代需要时间**与 map 的size 成比**，而**不关其capacity**。迭代HashMap可能会更加昂贵，需要的时间**与其capacity成正比**。



LinkedHashMap 有两个影响其性能的参数：初始容量和负载因子。它们的定义与HashMap完全相同。但请注意，对于此类，初始容量选择过高值对性能的影响不会像HashMap那么严重，因为此类的迭代时间不受容量影响，迭代只受到 size 的影响。



## 三、并发问题（fail-fast）

并发方式：推荐使用 `   Map m = Collections.synchronizedMap(new LinkedHashMap(...));`进行包装。内部是使用一个内置锁，各个方法的实现只是加锁然后 调用原来的类。这最好在创建时就完成「包装操作」，以防止意外的不同步访问Map。



结构性修改（A structural modification ） 是指

- **添加或删除**一个或多个映射的操作，
- 在访问有序的LinkedHashMap的情况下会影响迭代顺序。
- 在插入有序的LinkedHashMap 中，仅更改与已包含在映射中的键相关的值不是结构修改。
- **在 access-ordered 模式下的 LinkedHashMap中，仅通过get查询map就是一种结构修改**。 

迭代遍历过程中如果发现结构性修改问题（使用 iterater#remove 方法产生的结构性修改问题除外 ），马上抛出ConcurrentModificationException，以免造成更大的损失。

「快速失败」具体实现是怎么样的呢？每趟遍历完成之后都会调用 checkForComodification 方法进行检查

java.util.ArrayList.Itr#checkForComodification

```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```



## 四、实现

### 节点数据结构

主要基于HashMap的节点数据结构实现。

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    //每个节点包含两个指针，指向前继节点与后继节点
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

双向链表实现的LinkedHshMap，所以每个节点须在HashMap的基础上添加指向前继节点与后继节点指针：before，after。



### 核心方法

HashMap 中定义了三个回调方法供 LinkedHashMap 重写。

```java
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

LinkedHashMap继承于HashMap，重新实现了这3个函数，顾名思义这三个函数的作用分别是：**节点访问后、节点插入后、节点移除后做一些事情**。



#### 1.afterNodeRemoval

双向链表删除节点可以参考这样思路。

假设 before —>   p —> after

要删除 p

##### 1.先处理 after 方向

if  p 为第一个节点 before == null  head = after

else before.after = after

##### 2.再处理 before 方向

if p 为最后一个节点  `after == null`   `tail = before`
else  after.before = before;



```java
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

#### 2.afterNodeInsertion

 evict参数有什么用？

- if false, the table is in creation mode.
  - Creation mode 是什么？刚刚创建时的模式？

HashMap#的 put 方法，调用 putVal 方法时，传入的 evit 为 true。 

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```



调用处传递 evict 全部为 true。

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```





#### 3.afterNodeAccess

与模式相关 `final boolean accessOrder`

只能在构造函数中指定，默认为 false，

- true 表示按访问顺序排序
- false 表示按插入顺序排序

把节点移到链表末尾。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {//如果是按照访问顺序，并且 不是最后一个元素
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```



### 4.put

**put** `put`函数在`LinkedHashMap`中未重新实现，只是实现了`afterNodeAccess`和`afterNodeInsertion`两个回调函数。

### 5.get

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

`get`函数重新实现并加入了`afterNodeAccess`来保证访问顺序

## 五、小结

1. 怎样保证插入顺序？ 使用前驱和后继指针，使得原来的HashMap有序，在`LinkedHashMap`中覆盖`HashMap`中`newNode` 方法，使得每次put数据时，新建的节点都是`LinkedHashMap.Entry<K,v>` 类型的，比普通的`HsahMap.Entry` 多一个前驱结点和一个后继节点，使用前驱和后继保证插入有序。
2. 怎么样保证访问顺序？ 覆盖父类`HashMap`的`afterNodeAccess` 方法，使得每次访问后，都改变链表顺序。使得原链表按访问排序。将最新一次访问的节点放到链表的最后。

## 六、参考资料与学习资源推荐

- [重识java-LinkedHashMap](https://my.oschina.net/hgfdoing/blog/634125)
- [官方文档](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html)



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！