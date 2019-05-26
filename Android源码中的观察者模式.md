---
title: Android源码中的观察者模式
date: 2017-08-09 09:20:49
tags: 
- 原理分析
- Android 进阶
- 设计模式
categories:
- 原理分析
- Android 进阶
- 设计模式
---
# 解决、解耦的钥匙——观察者模式

## 定义
观察者模式定义了对象间一种**一对多**的依赖关系，使得每当一个对象改变状态，则所有依赖于它的对象都会得到通知并被自动更新。


## 使用场景
- 关联行为场景
    - 需要注意的是，关联行为是可拆分的，而不是“组合”关系
- 事件多级触发场景
- 跨系统的消息交换场景，如消息队列、事件总线的处理机制。


<!--more-->

## UML 类图 


UML 类图如下所示：
![uml](https://user-images.githubusercontent.com/16668676/29015194-a1c0db10-7b7f-11e7-99a0-437680f88188.png)


四个角色：
- Subject 抽象主题（抽象的被观察者）。把所有观察者对象的引用保存在一个集合里，每个主题可以有任意数量的观察者，提供接口供添加和删除观察者对象。
- ConcreteSubject 具体主题（具体的被观察者）。将有关状态存入具体观察者对象，在具体主题的内部实现发生改变时，给所有注册过的观察者发出通知
- Observer 抽象观察者。定义一个更新接口，用来接收主题更改时做出相应的操作（比如更新自身的状态）
- ConcreteObserver 具体观察者（实现抽象观察者所定义的更新接口，以便在主题状态更改时更新自身的状态）




## Android ListView 的观察者模式
ListView 和 RecyclerView 在 Android 中都占据着很重要的地位。

使用这两个组件时，总免不了数据更新的情况：当要更新数据时我们通常都会调用 `Adapter.notifyDataSetChanged()`，这其中的原理又是怎么样的呢？

下面以 ListView 作为一个栗子，看看其中的具体实现是怎么样的。


```
public void notifyDataSetChanged() {
    mDataSetObservable.notifyChanged();
}
```
```
public void notifyChanged() {
    synchronized(mObservers) {
        // since onChanged() is implemented by the app, it could do anything, including
        // removing itself from {@link mObservers} - and that could cause problems if
        // an iterator is used on the ArrayList {@link mObservers}.
        // to avoid such problems, just march thru the list in the reverse order.
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onChanged();
        }
    }
}
```
- notifyDataSetChanged 方法会调用  `DataSetObservable.notifyChanged()` 方法,notifyChanged() 内部会遍历调用观察者的 onChanged() 方法。通知数据更新。
- 但是观察者又是什么时候注册的呢？

以下为 setAdapter 的方法的具体实现：
```
@Override
public void setAdapter(ListAdapter adapter) {
    //如果 已经有 Adapter 存在，先解除注册
    if (mAdapter != null && mDataSetObserver != null) {
        mAdapter.unregisterDataSetObserver(mDataSetObserver);
    }

    resetList();
    mRecycler.clear();

    if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
        mAdapter = wrapHeaderListAdapterInternal(mHeaderViewInfos, mFooterViewInfos, adapter);
    } else {
        mAdapter = adapter;
    }

    mOldSelectedPosition = INVALID_POSITION;
    mOldSelectedRowId = INVALID_ROW_ID;

    // AbsListView#setAdapter will update choice mode states.
    super.setAdapter(adapter);

    if (mAdapter != null) {
        mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
        mOldItemCount = mItemCount;
        mItemCount = mAdapter.getCount();
        checkFocus();
        // 构建一个 AdapterDataSetObserver 
        mDataSetObserver = new AdapterDataSetObserver();
        mAdapter.registerDataSetObserver(mDataSetObserver);//注册观察者

        mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

        int position;
        if (mStackFromBottom) {
            position = lookForSelectablePosition(mItemCount - 1, false);
        } else {
            position = lookForSelectablePosition(0, true);
        }
        setSelectedPositionInt(position);
        setNextSelectedPositionInt(position);

        if (mItemCount == 0) {
            // Nothing selected
            checkSelectionChanged();
        }
    } else {
        mAreAllItemsSelectable = true;
        checkFocus();
        // Nothing selected
        checkSelectionChanged();
    }

    requestLayout();
}

```

通过源码可以看到，setAdapter 方法内部会构建一个 `AdapterDataSetObserver` ，然后注册为观察者。我们还可以看到注册的时候是通过 Adapter 来注册的。

Adapter 接口中声明了注册和解注册的方法签名。
```
public interface Adapter {
    /**
     * Register an observer that is called when changes happen to the data used by this adapter.
     *
     * @param observer the object that gets notified when the data set changes.
     */
    void registerDataSetObserver(DataSetObserver observer);

    /**
     * Unregister an observer that has previously been registered with this
     * adapter via {@link #registerDataSetObserver}.
     *
     * @param observer the object to unregister.
     */
    void unregisterDataSetObserver(DataSetObserver observer);
    //代码省略
}
```

而 BaseAdapter 实现了 ListAdapter 接口，ListAdapter 接口又继承自 Adapter 接口。
BaseAdapter 中注册方法和解除注册方法的具体实现：

```
public void registerDataSetObserver(DataSetObserver observer) {
    mDataSetObservable.registerObserver(observer);
}

public void unregisterDataSetObserver(DataSetObserver observer) {
    mDataSetObservable.unregisterObserver(observer);
}
```
BaseAdapter 中注册方法和解除注册方法的具体实现都是通过 mDataSetObservable 来完成的。查看一下 DataSetObservable 的具体实现。
```
public class DataSetObservable extends Observable<DataSetObserver> {
    public void notifyChanged() {
        synchronized(mObservers) {
            for (int i = mObservers.size() - 1; i >= 0; i--) {
                mObservers.get(i).onChanged();
            }
        }
    }

    public void notifyInvalidated() {
        synchronized (mObservers) {
            for (int i = mObservers.size() - 1; i >= 0; i--) {
                mObservers.get(i).onInvalidated();
            }
        }
    }
}
```
该类继承了 `android.database` 包下的 Observable 类，而 Observable 类中存有所有观察者的引用，以及注册和解注册方法。


ListView 中的 onChange 方法具体实现又是什么样的?
- 还记得前面我们说 Observer 是如何注册的吗？注册时，我们构建的是 AdapterDataSetObserver。
- 该类是 AbsListView 的内部类。
    - AbsListView.AdapterDataSetObserver 继承自 AdapterView<ListAdapter>.AdapterDataSetObserver 
    - onChange 方法的主要逻辑都在 AdapterDataSetObserver 中

```
class AdapterDataSetObserver extends DataSetObserver {
    //代码省略 ...
    @Override
    public void onChanged() {
        mDataChanged = true;
        mOldItemCount = mItemCount;
        mItemCount = getAdapter().getCount();

        // Detect the case where a cursor that was previously invalidated has
        // been repopulated with new data.
        if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
                && mOldItemCount == 0 && mItemCount > 0) {
            AdapterView.this.onRestoreInstanceState(mInstanceState);
            mInstanceState = null;
        } else {
            rememberSyncState();
        }
        checkFocus();
        requestLayout();
    }
    //代码省略 ...
}
```
onChange 方法中，会获取当前 Adapter 中的数据集的新数量，方法的最后调用 ListView 的 requestLayout() 方法重新进行布局。



从上面的分析中，我们可以看到
- AbsListView 是抽象的观察者
- ListView 是具体的观察者
- Adapter 接口是抽象的被观察者
- BaseAdapter 是具体的被观察者，其内部实际上是通过 `android.database` 包下的 Observerable 来实现注册和监听的。

### 小结
- AdapterView 中有一个 AdapterDataSetObserver 内部类，
- 在 ListView 中设置 Adapter（即调用 ListView 的 setAdapter 方法）时，其内部会构建一个 AdapterDataSetObserver，并注册到 Adapter 中。可见该场景中 ListView 扮演一个观察者角色。
- 而 Adapter 中有一个数据集可观察者 DataSetObserable。即一个被观察者。
- 数据集发生变化时，开发者手动调用 Adapter.notifyDataSetChanged 方法，notifyDataSetChanged 会调用 `DataSetObserverable.notifyChanged()`
    - notifyChanged() 方法会遍历所有观察者，并调用观察者的 `onChanged` 方法，
    - onChanged 会获取 Adapter 中数据集的新数量，同时会调用 View.requestLayout 方法通知 ListView 更新布局，刷新用户界面。

虽然 RecyclerView 直接继承自 ViewGroup，而不是 AdapterView ，但是更新列表数据集方法的内部也是通过观察者模式来实现的。只是在其中多提供了单个数据或者指定范围数据更新等回调接口，供开发者使用。感兴趣的同学可以去看看 RecyclerView 中观察者模式的实现。


作者水平有限，疏漏之处，恳请指出。

## 参考 
- 《Android 源码设计模式解析与实战》 第十二章 