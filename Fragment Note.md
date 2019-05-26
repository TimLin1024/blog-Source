---
title: Fragment Note
date: 2018-06-02 16:13:05
tags: 
- Android
categories:
- Android
---



笔记自用，部分内容待完善。

## 概述

当您将片段作为 Activity 布局的一部分添加时，它存在于 Activity 视图层次结构的某个 `ViewGroup` 内部，并且**片段会定义其自己的视图布局**。您可以通过在 Activity 的**布局文件中声明片段**，将其作为 `<fragment>` 元素插入您的 Activity 布局中，或者通过将其添加到某个现有 `ViewGroup`，利用应用代码进行插入。不过，片段**并非必须成为 Activity 布局的一部分**；您还可以将没有自己 UI 的片段用作 Activity 的不可见工作线程。

<!--more-->



### Fragment 解决了什么问题？

不能直接用 View 替代吗？

相对于 view 而言，它的优势在哪？

 Android 3.0（API 级别 11）中引入了片段，主要是为了给大屏幕（如平板电脑）上更加动态和灵活的 UI 设计提供支持。由于平板电脑的屏幕比手机屏幕大得多，因此可用于组合和交换 UI 组件的空间更大。利用片段实现此类设计时，您**无需管理对视图层次结构的复杂更改**。 

您应该将每个片段都设计为**可重复使用的模块化 Activity 组件**。也就是说，由于每个片段都会通过各自的生命周期回调来定义其自己的布局和行为，您可以**将一个片段加入多个 Activity**，因此，您**应该采用可复用式设计，避免直接从某个片段直接操纵另一个片段**。 这特别重要，因为**模块化片段让您可以通过更改片段的组合方式来适应不同的屏幕尺寸**。 在设计可同时支持平板电脑和手机的应用时，您可以在不同的布局配置中重复使用您的片段，以根据可用的屏幕空间优化用户体验。 

## 使用

通常至少需要实现以下几个生命周期方法：

- onCreate（）
  - 在此方法中初始化那些你希望在fragment 处于 paused or stopped, then resumed状态的时候仍然被保留的组件

- onCreateView()
  - 这个方法必须返回 这个fragment的布局的根View，也如果此fragment不提供一个UI的话，可以返回空

- onPause()
  - 这个方法中你应该**提交你的fragment中的那些需要被保存的数据**。



### 参数传递

使用 setArgument 的方式，通过这种方式传递参数，如果配置发生更改，通过`Fragment.setArguments(Bundle bundle)`方法设置的bundle会保留下来。

如果是通过直接赋值的方式，则需要实现 onSaveInstance 和 onRestoreIntanceState 方法（或者 在 onCreate 方法进行配置）才能应对配置更改。

要注意的问题就是 setArgument 方法要在 Fragment 与 Activity 关联之前调用。





### 可以被继承的Fragment基类

- DialogFragment

> Displays a floating dialog. Using this class to create a dialog is a good alternative to using the dialog helper methods in the Activity class, because you can incorporate(包含) a fragment dialog into the back stack of fragments managed by the activity, allowing the user to return to a dismissed fragment.



相比于 Dialog 的好处：

1. 生命周期便于管理；
2. 比 Dialog 更加地强大。「DialogFragment也允许开发者把Dialog作为内嵌的组件进行重用」

重写 onCreateView 或者  onCreateDialog 





- ListFragment

> Displays a list of items that are managed by an adapter (such as a SimpleCursorAdapter), similar to ListActivity. It provides several methods for managing a list view, such as the onListItemClick() callback to handle click events.

类似于ListActivity

- PreferenceFragment

> Displays a hierarchy of Preference objects as a list, similar to PreferenceActivity. This is useful when creating a "settings" activity for your application.

对于“设置”界面而言很有用



### 特殊用途

调用 `setRetainInstance(true);`，会在配置发生改变的时候保存当前Fragment 中的所有对象。

```java
/**
 * Control whether a fragment instance is retained across Activity
 * re-creation (such as from a configuration change).  This can only
 * be used with fragments not in the back stack.  If set, the fragment
 * lifecycle will be slightly different when an activity is recreated:
 * <ul>
 * <li> {@link #onDestroy()} will not be called (but {@link #onDetach()} still
 * will be, because the fragment is being detached from its current activity).
 * <li> {@link #onCreate(Bundle)} will not be called since the fragment
 * is not being re-created.
 * <li> {@link #onAttach(Activity)} and {@link #onActivityCreated(Bundle)} <b>will</b>
 * still be called.
 * </ul>
 */
该方法控制 Fragment 的实例是否在Activity重新创建的时候保留。
它只能用于那些不处在回退栈中的Fragment 身上。
注意，如果保留实例的话，Fragment 的生命周期将会有一些不同
- onDestory 方法不会被调用，但是 onDetach 方法仍然会被调用，因为 Fragment 从当前 Activity 中剥离了。
- onCreate 方法也不会被调用，因为 Fragment 并没有重新创建
- onAttach 和 onActivityCreated 方法依然会被调用。
public void setRetainInstance(boolean retain) {
    mRetainInstance = retain;
}
```



用于保存数据。







## 创建/渲染用户界面

为了提供一个布局给fragment，你必须实现onCreateView()方法,这个方法会在到这个Fragment 绘出它的布局的时候 被安卓系统调用。
此外，这个方法中必须 return a View that is the root of your fragment's layout

> Note: If your fragment is a subclass of ListFragment, the default implementation returns a ListView from onCreateView(), so you don't need to implement it.



`inflate()` 方法带有三个参数：

- 您想要inflate的布局的资源 ID；
- 将作为inflate布局父项的 `ViewGroup`。传递 `container` 对系统向inflate布局的根视图（由其所属的父视图指定）应用布局参数具有重要意义；
- 指示是否应该在inflate期间将inflate布局附加至 `ViewGroup`（第二个参数）的布尔值。（在本例中，其值为 false，因为系统已经将inflate布局插入 `container` — 传递 true 值会在最终布局中创建一个多余的视图组。）

系统会帮我们 addView，所以 attachToRoot 为 false；

`android.support.v4.app.FragmentManagerImpl#moveToState(android.support.v4.app.Fragment, int, int, int, boolean)`

```java
if(container != null) {
    container.addView(f.mView);
}
```



推荐用下边这种方式：

```
inflater.inflate(R.layout.item, parent, false);
```

测量的时候依赖 parent，但是添加需要手动进行

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            // Look for the root node.
            int type;
            final String name = parser.getName();

            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    //创建 根布局的 layout params
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }

                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);

                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }


                //根布局存在 并且 待渲染的布局需要关联到 root，那就调用 addView 方法将它添加进去
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                //确定返回值是 root 还是  xml 文件中的根布局
                //root 为 null 或者 不关联到根布局，返回的就是 xml 文件中的根布局
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            //……
        }
        return result;
    }
}
```



## 将 Fragment 添加到 Activity 的两种方式

### 1.在 Activity 的布局文件内声明片段

- 当系统创建此 Activity 布局时，会**实例化在布局中指定的每个片段**，并为每个片段调用 `onCreateView()` 方法，以检索每个片段的布局。**系统会直接插入片段返回的 `View` 来替代 `<fragment>` 元素**。

`<fragment>` 元素中可以通过 `class = “ ”` 也可以通过 `android:name=""` 来指定 要加入的Fragment。

class 的性能会高一点点。

因为：android.support.v4.app.FragmentManagerImpl#onCreateView(android.view.View, java.lang.String, android.content.Context, android.util.AttributeSet)

```java
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    if (!"fragment".equals(name)) {
        return null;
    }
	//通过 class 获取 Fragment 名字
    String fname = attrs.getAttributeValue(null, "class");
    TypedArray a =  context.obtainStyledAttributes(attrs, FragmentTag.Fragment);
    if (fname == null) {//用 class 找不到，通过 android:name 去查找
        fname = a.getString(FragmentTag.Fragment_name);
    }
	//……
}
```



**注**：每个片段都需要一个唯一的标识符，重启 Activity 时，系统可以使用该标识符来恢复片段（您也可以使用该标识符来捕获片段以执行某些事务，如将其移除）。 可以通过三种方式为片段提供 ID：

- 为 `android:id` 属性提供唯一 ID。
- 为 `android:tag` 属性提供唯一字符串。
- 如果您未给以上两个属性提供值，系统会使用容器视图的 ID。

### 2.通过java 代码将片段添加到某个现有 ViewGroup

您可以在 Activity 运行期间随时将片段添加到 Activity 布局中。您只需指定要将片段放入哪个 `ViewGroup`。

您必须使用 `FragmentTransaction` 中的 API,才能在 Activity 中执行片段事务（如添加、移除或替换片段）。您可以像下面这样从 `Activity` 获取一个 `FragmentTransaction` 实例：

```java
FragmentManager fragmentManager = getFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```

然后，您可以使用 `add()` 方法添加一个片段，**指定要添加的片段以及将其插入哪个视图**。例如：

```java
ExampleFragment fragment = new ExampleFragment();
fragmentTransaction.add(R.id.fragment_container, fragment);//将要添加的片段
fragmentTransaction.commit();
```

**传递到 `add()` 的第一个参数是 `ViewGroup`，即应该放置片段的位置，由资源 ID 指定，第二个参数是要添加的片段**。

一旦您通过 `FragmentTransaction` 做出了更改，就必须调用 `commit()` 以使更改生效。



#### 添加没有 UI 的片段

可以使用片段为 Activity 提供后台行为，而不显示额外 UI。

要想添加没有 UI 的片段，请使用 `add(Fragment, String)` 从 Activity 添加片段（为片段提供一个唯一的字符串“标记”，而不是视图 ID）。 这会添加片段，但**由于它并不与 Activity 布局中的视图关联，因此不会收到对`onCreateView()` 的调用**。因此，您不需要实现 `onCreateView()`方法。

如果片段没有 UI，则字符串标记将是标识它的唯一方式。



### 管理片段

要想管理您的 Activity 中的片段，您需要使用 `FragmentManager`。要想获取它，请从您的 Activity 调用 `getFragmentManager()`。

您可以使用 `FragmentManager` 执行的操作包括：

- 通过 `findFragmentById()`（对于在 Activity 布局中提供 UI 的片段）或 `findFragmentByTag()`（对于提供或不提供 UI 的片段）获取 Activity 中存在的片段。
- 通过 `popBackStack()`（模拟用户发出的*返回*命令）将片段从返回栈中弹出。
- 通过 `addOnBackStackChangedListener()` 注册一个监听返回栈变化的侦听器。

如需了解有关这些方法以及其他方法的详细信息，请参阅 `FragmentManager` 类文档。

如上文所示，您也可以使用 `FragmentManager` 打开一个 `FragmentTransaction`，通过它来执行某些事务，如添加和移除片段。



## 执行片段事务

在 Activity 中使用片段的一大优点是，可以根据用户行为通过它们执行添加、移除、替换以及其他操作。 您**提交给 Activity 的*每组更改都称为事务***，您**可以使用 `FragmentTransaction` 中的 API 来执行一项事务**。您也可以将每个事务保存到由 Activity 管理的返回栈内，从而让用户能够回退片段更改（类似于回退 Activity）。

可以从 `FragmentManager` 获取一个 `FragmentTransaction` 实例。



**每个事务**（从FragmentTransaction# begin 到 commit 之间的一组「添加，删除，隐藏，显示」操作）都是您想要同时执行的**一组更改**。您可以使用 `add()`、`remove()` 和 `replace()` 等方法为给定事务设置您想要执行的所有更改。然后，要想将事务应用到 Activity，您必须调用 `commit()`。

不过，在您调用 `commit()` 之前，您可能想**调用 `addToBackStack()`，以将事务添加到片段事务返回栈**。 该返回栈由 Activity 管理，允许用户通过按*返回*按钮返回上一片段状态。

例如，以下示例说明了如何将一个片段替换成另一个片段，以及如何在返回栈中保留先前状态：

```java
// Create new fragment and transaction
Fragment newFragment = new ExampleFragment();
FragmentTransaction transaction = getFragmentManager().beginTransaction();

// Replace whatever is in the fragment_container view with this fragment,
// and add the transaction to the back stack
transaction.replace(R.id.fragment_container, newFragment);
transaction.addToBackStack(null);

// Commit the transaction
transaction.commit();
```

在上例中，`newFragment` 会替换目前在 `R.id.fragment_container` ID 所标识的布局容器中的任何片段（如有）。通过**调用 `addToBackStack()` 可将替换事务保存到返回栈**，以便用户能够通过按*返回*按钮撤消事务并回退到上一片段。

- 替换事务 指的是什么？

如果您向事务添加了多个更改（如又一个 `add()` 或 `remove()`），并且调用了 `addToBackStack()`，则在调用 `commit()` 前应用的**所有更改都将作为单一事务**添加到返回栈，并且*返回*按钮会将它们一并撤消。

向 `FragmentTransaction` 添加更改的顺序无关紧要，不过：

- 您必须最后调用 `commit()`
- 如果您要向同一容器添加多个片段，则您添加片段的顺序将决定它们在视图层次结构中的出现顺序

如果您**没有在执行移除片段的事务时调用 `addToBackStack()`，则事务提交时该片段会被销毁**，用户将无法回退到该片段。 不过，如果您在删除片段时调用了 `addToBackStack()`，则系统会*停止*该片段，并在用户回退时将其恢复。

**提示**：对于每个片段事务，您都可以通过**在提交前调用 `setTransition()` 来应用过渡动画**。

调用 `commit()` **不会立即执行事务**，而是在 Activity 的 UI 线程（“主”线程）可以执行该操作时再安排其在线程上运行。不过，如有必要，您**也可以从 UI 线程调用 `executePendingTransactions()` 以立即执行 `commit()` 提交的事务**。通常不必这样做，除非其他线程中的作业依赖该事务。

**注意**：您只能在 Activity 保存其状态（用户离开 Activity）之前使用 `commit()` 提交事务。如果您试图在该时间点后提交，则会引发异常。 这是因为如需恢复 Activity，则提交后的状态可能会丢失。 对于丢失提交无关紧要的情况，请使用 `commitAllowingStateLoss()`。



### FragmentTransaction#commit vs  FragmentTransaction#commitAllowingStateLoss



### 回退栈



FragmentTransaction 是在一个链表中存储了一次事务中的所有需要执行的操作。

```java
/**
 * Add this transaction to the back stack.  This means that the transaction
 * will be remembered after it is committed, and will reverse its operation
 * when later popped off the stack.
 *
 * @param name An optional name for this back stack state, or null.
 */
//添加「事务」注意不是某一个 Fragment到回退栈中。这意味着这个事务会在提交后被「记住」
public abstract FragmentTransaction addToBackStack(@Nullable String name);
```

BackStackRecod 本身还是实现了 Runnabled 接口，是一个可以执行的对象。添加完毕之后，会调用 FragmentHostCallback 中提供的 getHandler 方法，获取到 Handler 方法，然后向主线程的 MessageQueue 中发送一个 mExecCommit 可执行对象

### 处理片段生命周期

![image](http://androiddoc.qiniudn.com/images/fragment_lifecycle.png)





![activity_fragment_lifecycle](https://ws1.sinaimg.cn/large/006tKfTcgy1fpze8kvyyuj309g0irdgj.jpg)

Activity 生命周期与片段生命周期之间的最显著差异在于它们在其**各自返回栈中的存储方式**。 默认情况下，Activity 停止时会被放入**由*系统管理的 Activity 返回栈***（以便用户通过*返回*按钮回退到 Activity，[任务和返回栈](https://developer.android.com/guide/components/tasks-and-back-stack.html)对此做了阐述）。不过，**仅当在移除片段的事务执行期间通过调用 `addToBackStack()` 显式请求保存实例时，系统才会将片段放入由*宿主 Activity 管理的返回栈***。

**在其他方面，管理片段生命周期与管理 Activity 生命周期非常相似**。 因此，[管理 Activity 生命周期](https://developer.android.com/guide/components/activities.html#Lifecycle)的做法同样适用于片段。 

**注意**：如果需要 Context 的话，可以调用 `getActivity()`。但要注意，请仅在片段附加到 Activity 时调用 `getActivity()`。如果片段尚未附加，或在其生命周期结束期间分离，则 `getActivity()` 将返回 null。



不过，片段还有几个额外的生命周期回调，用于处理与 Activity 的唯一交互，以执行构建和销毁片段 UI 等操作。 这些**额外的回调方法**是：

- `onAttach()`

  在片段已与 Activity 关联时调用（`Activity` 传递到此方法内）。

- `onCreateView()`

  调用它可创建与片段关联的视图层次结构。

- `onActivityCreated()`

  在 Activity 的 `onCreate()` 方法已返回时调用。

- `onDestroyView()`

  在移除与片段关联的视图层次结构时调用。

- `onDetach()`

  在**取消片段与 Activity 的关联时**调用。

一旦 Activity 达到 resume 状态，您就可以随意向 Activity 添加片段和移除其中的片段。 因此，**只有当 Activity 处于 resume 状态时，片段的生命周期才能独立变化**。

Activity 处于resume 状态下时，创建一个 Fragment ，这个Fragment 前面的生命周期会快速走完，直到 resume

不过，当 Activity 离开 resume 状态时，片段会在 Activity 的推动下再次经历其生命周期。







## 与 Activity 对比

### 构造方法

普通开发者无权使用 Activity 的构造方法，所以，不能通过构造方法从外部注入依赖。

Fragment 的构造方法可以传参进来

Activity 一般是通过 intent。Fragment 会用 setArgument 传一个 Bundle 进来。



### 设置布局&初始化控件的时机不同

`Activity#onCreate`

`Fragment#onCreateView`



## 原理

- 注: FragmentManagerImpl 定义在 FragmentManager的类文件里面，直接搜只能看到 .class,找不到 .java 文件

起始：查找某一个类只看到 .class文件的时候，可以先看看 .class ，它继承/实现了哪些类，然后到父类那里去找，一般都可以找到



我们常看到说 Fragment 的生命周期会与宿主Activity 保持一致，里面究竟是如何实现的呢？

Fragment 的生命周期会在 FragmentActivity 里面的生命周期进行回调



### 生命周期

因为 Fragment 依附于 Activity（生命周期状态由 Activity 控制），所以Fragment 的生命周期方法是在 Activity 的生命周期方法之后才调用的。

android.app.FragmentManagerImpl#onCreateView()

```java
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {    
    //……
    
    // If we haven't finished entering the CREATED state ourselves yet,
    // push the inflated child fragment along. This will ensureInflatedFragmentView
    // at the right phase of the lifecycle so that we will have mView populated
    // for compliant fragments below.
    if (mCurState < Fragment.CREATED && fragment.mFromLayout) {
        moveToState(fragment, Fragment.CREATED, 0, 0, false);
    } else {
        moveToState(fragment);
    }
    
    //……
}
```





### 提交事务

#### add hide show replace

当然我们注意到 hide 和 show，它们内部实际上不会调用 moveToState, 因为 **hideFragment 实际上就做了三件事请**，

- 设置 mHidden 为 true
- fragment.mView.setVisibility(View.GONE); 隐藏 fragment 的 view
- fragment.onHiddenChanged(true); 调用 onHiddenChange

showFragment 也是类似，只不过行为正好相反。



#### addFragment

addFragment 会将 fragment 添加到 mAdded 和 mActive 这两个集合当中，这两个集合维护了当前 activity 中维护的已经添加的 fragment 列表和当前处于活跃状态的 fragment 列表，如果 fragment 位于 mActive 中，那么当 activity 的状态发生变化时，fragment 也会跟随着发生变化。FragmentManger 如何引导 fragment 的状态发生变化呢？



#### Fragment 状态变迁：moveToState

Fragment 状态变迁发生在用户主动发起 transaction，或者 fragment 被 add 到 activity 之后跟随 activity 的生命周期变化一起发生改变。每次状态变迁最终都会走到函数 moveToState，字面意思是将 fragment 迁移到新的状态



fragment 的 state 取值，为前面提到的七种状态，其中最低值是 INITIALIZING 状态，代表 fragment **刚创建，还未被 add**， 最高状态值是 RESUMED, 代表 fragment 处于前台。 所以 moveToState 内部分两条线，状态跃升，和状态降低，里面各有一个 switch 判断，注意到 **switch 里每个 case 都没有 break，这意味着，状态可以持续变迁**，比如从 INITIALIZING，一直跃升到 RESUMED，将每个 case 都走一遍，每次 case 语句内，都会改变 state 的值。



![img](https://upload-images.jianshu.io/upload_images/1008428-c0809de4a37a0d74.jpg)





```
mManager.moveToState(mManager.mCurState, transition, transitionStyle, true);
```

就是将 fragment 迁移到 FragmentManager 当前的状态，因为我们不知道用户什么时候 add fragment，因此 fragment 被 add 之后，就将其状态迁移到 FragmentManager 当前的状态，然后跟随 FragmentManager 一起发生状态变迁，除非用户手动 removeFragment 将其从 mActive 列表中移除。









```java
getSupportFragmentManager()
                        .beginTransaction()
                        .add(,)
                        .commit();
```

```java
FragmentManagerImpl#beginTransaction //创建一个  BackStackRecord;
	--> BackStackRecord#add(Fragment, String)
		--> BackStackRecord#doAddOp
			--> //进行访问控制检查，如果 Fragment 是一个成员类，那么它必须是 public static 的，这样才能正确地从实例状态中重新创建出来
				给 mFragmentManager、 mTag 、mContainerId  赋值
				创建 Op 对象并添加到 Ops （一个 ArrayList） 中 				
			--> BackStackRecord#commit
			--> BackStackRecord#commitInternal
				if mAddToBackStack //添加到回退栈中
                      mIndex = mManager.allocBackStackIndex(this);//分配回退栈下标
						--> 主要对下面两个列表进行操作
						ArrayList<BackStackRecord> mBackStackIndices;
    					ArrayList<Integer> mAvailBackStackIndices;
                  else 
                      mIndex = -1;
                  
					--> FragmentManagerImpl#enqueueAction//在等待操作的队列中添加一个操作。
                      	-->  mPendingActions.add(action);//添加到「队列中」 （实际上是一个 ArrayList）
            		    --> scheduleCommit();//安排 commit
							--> 内部通过 Handler 来实现
								执行的是 mExecCommit （一个 Runnable）
								--> FragmentManagerImpl#execPendingActions
```



FragmentManagerImpl#execPendingActions //只能从主线程调用

```java
public boolean execPendingActions() {
    ensureExecReady(true);

    boolean didSomething = false;
    while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
            optimizeAndExecuteOps(mTmpRecords, mTmpIsPop);
        } finally {
            cleanupExec();
        }
        didSomething = true;
    }

    doPendingDeferredStart();
    burpActive();

    return didSomething;
}
```





## 参考资料与学习资源推荐

- https://www.jianshu.com/p/180d2cc0feb5

- [Fragment | Android Developers](https://developer.android.com/reference/android/app/Fragment)



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！