---
title: Skeleton 工作原理
date: 2018-11-07 00:32:15
tags:
categories:
- 框架原理
---



## 1.简介

Skeleton Screen（加载占位图）是近年流行的加载控件，通常表现形式是在界面上待加载区域填充灰色的占位图，与线框图的效果非常相似。Skeleton Screen 本质上是界面加载过程中的过渡效果。

<!--more-->

使用 Skeleton 的效果图如下：

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fwyt63fc6bj30fz0sgwes.jpg)







## 2.工作原理

从上面的效果图，不难看出，「Skeleton」有一个显示和隐藏的过程。下面针对普通的 view 和 RecyclerView 分别说明其中的显示/隐藏的过程。

### 2.1 对于普通的 View

从常规的使用方法看起。

```java
mTvSkeletonScreen = Skeleton.bind(textView)
        .load(R.layout.share_skeleton_text_view)
        .show();
```





首先调用了 `Skeleton.bind();`

```java
public static ViewSkeletonScreen.Builder bind(View view) {
    return new ViewSkeletonScreen.Builder(view);
}
```

bind 方法返回的是一个 ViewSkeletonScreen.Builder

也就是一个建造器，其中提供了各种配置方法。

```java
public static class Builder {
    private final View mView;//依附的 view
    private int mSkeletonLayoutResID;//外部注入的 id
    private boolean mShimmer = true;//是否闪烁
    private int mShimmerColor;//闪烁的颜色
    private int mShimmerDuration = 1000;//闪烁时间，默认是 1000ms
    private int mShimmerAngle = 20;//闪烁的角度

    public Builder(View view) {
        this.mView = view;//从外部注入的目标 view
        this.mShimmerColor = ContextCompat.getColor(mView.getContext(), R.color.shimmer_color);//默认的 shimmer 颜色
    }
	
    //代码省略
    
    public ViewSkeletonScreen show() {
        ViewSkeletonScreen skeletonScreen = new ViewSkeletonScreen(this);
        skeletonScreen.show();
        return skeletonScreen;
    }

}
```



设置好之后，调用 show，即可将 Skeleton 显示 出来

ViewSkeletonScreen.Builder#show

```java
public ViewSkeletonScreen show() {
    ViewSkeletonScreen skeletonScreen = new ViewSkeletonScreen(this);//根据当前 builder 创建一个 ViewSkeletonScreen
    skeletonScreen.show();//显示
    return skeletonScreen;
}
```



Builder#show 方法返回的是一个 ViewSkeletonScreen 对象。

ViewSkeletonScreen 实现了 SkeletonScreen 接口。

```java
public interface SkeletonScreen {

    void show();

    void hide();
}
```



#### 2.1.1 显示

##### ViewSkeletonScreen#show() 

```java
public void show() {
    View skeletonLoadingView = generateSkeletonLoadingView();//生成 loading view
    if (skeletonLoadingView != null) {
        mViewReplacer.replace(skeletonLoadingView);//将 bind 进来的 view 替换为 loadingview
    }
}
```



##### 1.生成 loadingView

```java
private View generateSkeletonLoadingView() {
    ViewParent viewParent = mActualView.getParent();//获取实际 view 的 parent
    if (viewParent == null) {//如果没有父 view
        Log.e(TAG, "the source view have not attach to any view");
        return null;
    }
    ViewGroup parentView = (ViewGroup) viewParent;
    if (mShimmer) {
        return generateShimmerContainerLayout(parentView);//生成 shimmer layout。在现有的 view 上面包装一层，当 view 关联到 window 的时候，开始播放动画；当 view 从 window 中分离的时候停止播放动画。 
    }
    return LayoutInflater.from(mActualView.getContext()).inflate(mSkeletonResID, parentView, false);//渲染出 view
}
```

##### 2.显示

```java
public void replace(View targetView) {
    if (mCurrentView == targetView) {
        return;
    }
    if (targetView.getParent() != null) {//如果 loadingview 的 parent 不为 null，将它从父容器中移除
        ((ViewGroup) targetView.getParent()).removeView(targetView);//移除 view
    }
    if (init()) {//初始化
        mTargetView = targetView;
        mSourceParentView.removeView(mCurrentView);//移除掉源 view
        mTargetView.setId(mSourceViewId);//设置 id，
        mSourceParentView.addView(mTargetView, mSourceViewIndexInParent, mSourceViewLayoutParams);//添加到父 view 中
        mCurrentView = mTargetView;
    }
}
```



判断条件

```java
private boolean init() {
    if (mSourceParentView == null) {//
        mSourceParentView = (ViewGroup) mSourceView.getParent();//获取 bind 进来的 view 的父容器
        if (mSourceParentView == null) {//如果父容器为空，说明未关联到任何 view，直接返回
            Log.e(TAG, "the source view have not attach to any view");
            return false;
        }
        //遍历获取当前 view 在父容器中下标
        int count = mSourceParentView.getChildCount();
        for (int index = 0; index < count; index++) {
            if (mSourceView == mSourceParentView.getChildAt(index)) {
                mSourceViewIndexInParent = index;
                break;
            }
        }
    }
    return true;
}
```





#### 2.1.2 隐藏

com.ethanhua.skeleton.ViewSkeletonScreen#hide

```java
@Override
public void hide() {
    if (mViewReplacer.getTargetView() instanceof ShimmerLayout) {
        ((ShimmerLayout) mViewReplacer.getTargetView()).stopShimmerAnimation();
    }
    mViewReplacer.restore();
}
```



com.ethanhua.skeleton.ViewReplacer#restore 

其实就是 replace 方法的逆向。将 「loading view 」从父 view 中移除掉，将「原 view」（bind 进来的 view），添加到父 view。

```java
public void restore() {
    if (mSourceParentView != null) {
        mSourceParentView.removeView(mCurrentView);//移除掉当前的 view。
        mSourceParentView.addView(mSourceView, mSourceViewIndexInParent, mSourceViewLayoutParams);
        mCurrentView = mSourceView;
        mTargetView = null;
        mTargetViewResID = -1;
    }
}
```













### 2.2 对于 RecyclerView



```java
public static RecyclerViewSkeletonScreen.Builder bind(RecyclerView recyclerView) {
    return new RecyclerViewSkeletonScreen.Builder(recyclerView);
}
```





```java
public static class Builder {
    private RecyclerView.Adapter mActualAdapter;//适配器
    private final RecyclerView mRecyclerView;
    private boolean mShimmer = true;//默认支持 shimmer 
    private int mItemCount = 10;//默认为 10
    private int mItemResID = R.layout.layout_default_item_skeleton;//默认的列表项布局
    private int mShimmerColor;//颜色
    private int mShimmerDuration = 1000;//
    private int mShimmerAngle = 20;
    private boolean mFrozen = true;

    public Builder(RecyclerView recyclerView) {
        this.mRecyclerView = recyclerView;
        this.mShimmerColor = ContextCompat.getColor(recyclerView.getContext(), R.color.shimmer_color);//默认的闪烁颜色
    }

    /**
     * @param adapter the target recyclerView actual adapter
     */
    public Builder adapter(RecyclerView.Adapter adapter) {
        this.mActualAdapter = adapter;
        return this;
    }

    /**
     * @param itemCount the child item count in recyclerView
     */
    public Builder count(int itemCount) {
        this.mItemCount = itemCount;
        return this;
    }
	//代码省略
    
    /**
     * @param frozen whether frozen recyclerView during skeleton showing
     * @return
     */
    public Builder frozen(boolean frozen) {
        this.mFrozen = frozen;
        return this;
    }

    public RecyclerViewSkeletonScreen show() {
        RecyclerViewSkeletonScreen recyclerViewSkeleton = new RecyclerViewSkeletonScreen(this);
        recyclerViewSkeleton.show();
        return recyclerViewSkeleton;
    }
}
```

与普通 view 不同， recyclerview 的 Skeleton Builder 提供了多个  adapter 方法来设置适配器，count 方法用于设置 列表项数目，frozen 控制 Skeleton 显示期间是否冻结 RecyclerView。



上面的 RecyclerViewSkeletonScreen.Builder#show 方法，首先创建了 一个 RecyclerViewSkeletonScreen ，然后调用它的 show 方法



RecyclerViewSkeletonScreen#RecyclerViewSkeletonScreen 

```java
private RecyclerViewSkeletonScreen(Builder builder) {
    mRecyclerView = builder.mRecyclerView;
    mActualAdapter = builder.mActualAdapter;//真正的适配器，隐藏 Skeleton 之后，显示真实的 view 时需要用到
    mSkeletonAdapter = new SkeletonAdapter();//创建 adapter
    mSkeletonAdapter.setItemCount(builder.mItemCount);//列表的总项数
    mSkeletonAdapter.setLayoutReference(builder.mItemResID);//列表项的 id
    mSkeletonAdapter.shimmer(builder.mShimmer);//是否闪烁
    mSkeletonAdapter.setShimmerColor(builder.mShimmerColor);//闪烁的颜色
    mSkeletonAdapter.setShimmerAngle(builder.mShimmerAngle);//闪烁的角度
    mSkeletonAdapter.setShimmerDuration(builder.mShimmerDuration);//闪烁时长
    mRecyclerViewFrozen = builder.mFrozen;//是否冻结
}
```



#### 2.2.1 显示

RecyclerViewSkeletonScreen#show

```java
public void show() {
    mRecyclerView.setAdapter(mSkeletonAdapter);
    if (!mRecyclerView.isComputingLayout() && mRecyclerViewFrozen) {//如果不是正在计算布局，并且设置了冻结，则将 Recyclerview 冻结
        mRecyclerView.setLayoutFrozen(true);
    }
}
```



#### 2.2.2 隐藏

RecyclerViewSkeletonScreen#hide

```java
@Override
public void hide() {
    mRecyclerView.setAdapter(mActualAdapter);//将真实的 adapter 设置回去。内部会触发刷新，进而显示出
}
```



SkeletonAdapter 的实现

```java
public class SkeletonAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        if (mShimmer) {
            return new ShimmerViewHolder(inflater, parent, mLayoutReference);
        }
        return new RecyclerView.ViewHolder(inflater.inflate(mLayoutReference, parent, false)) {
        };
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        if (mShimmer) {
            ShimmerLayout layout = (ShimmerLayout) holder.itemView;
            layout.setShimmerAnimationDuration(mShimmerDuration);
            layout.setShimmerAngle(mShimmerAngle);
            layout.setShimmerColor(mColor);
            layout.startShimmerAnimation();
        }
    }
	//代码省略
}
```

可以看到还是跟普通的 view 一个套路，不闪烁的话，直接将列表项渲染出来就完事。如果设置了闪烁，则需要包装到一个 ShimmerLayout 中，并根据参数进一步设置。





## 3.总结

普通 view， 核心思想就是，利用 ViewGroup#addView，ViewGroup#removeView 对 view 进行替换。

显示 Skeleton 的时候，将原来的 view 替换为 loadingView

隐藏 Skeleton 的时候，将「loadingview」替换为 sourceView。



对于 RecyclerView，在显示期间，通过 SkeletonAdapter 偷天换日，将 loading 显示出来。

隐藏的时候，将真正的 adapter 设置回去。



## 4.参考资料与学习资源推荐

- [这个控件叫：Skeleton Screen/加载占位图](https://zhuanlan.zhihu.com/p/26014116)
- [Skeleton](https://github.com/ethanhua/Skeleton)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！