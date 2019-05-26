---
title: 深入理解 LayoutInflater
date: 2017-08-17 18:08:20
tags: 
- 原理分析
- Android 进阶
categories:
- 原理分析
- Android 进阶

---



## 一、概述

 `View rootView  = LayoutInflater.from(getContext()).inflate(R.layout.bitable_grid_view, this, false);`

Android 开发者对上面这行代码应该不陌生，我们通常用这个方法来渲染一个布局。其中的原理是怎么样的呢？
本文主要分析 LayoutInflater 的创建过程以及 inflate 方法的原理。

<!--more-->

## 二、LayoutInflater 是如何创建的

从 `LayoutInflater#from ` 方法看起。

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

从代码中可以看出，`LayoutInflater` 是通过 `Context#getSystemService` 的方式获取的。



Context 是一个抽象类，它的具体实现类为 `ContextImpl`。所以应该看 ContextImpl 的 getSystemService 是如何实现的。

`ContextImpl#getSystemService`

```java
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

`ContextImpl#getSystemService` 方法又委托了 SystemServiceRegistry。



`SystemServiceRegistry` 中有一个 `HashMap<String, ServiceFetcher<?>>` ，该 HashMap 以服务名称为 key，以服务相对应的 ServiceFetcher 作为 value。

```java
private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
        new HashMap<>();
```

具体的获取方法为

```Java
/**
 * Gets a system service from a given context.
 */
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}
```

根据服务名称去获取相应的 ServiceFetcher，

- 如果 `ServiceFetcher` 不为空，则调用 `ServiceFetcher.getService` 方法获取相应服务的引用。
  - 如果是第一次调用会先创建，然后直接返回
  - 否则直接返回缓存的值
- 如果 `ServiceFetcher` 为空，则返回 null。



### SYSTEM_SERVICE_FETCHER 数据初始化

`SYSTEM_SERVICE_FETCHERS` 是一个键为 String, 值为 `ServiceFetcher<?>` 的静态 HashMap 常量，其中的数据是在静态代码块中插入的。

LayoutInflater 对应的 ServiceFetcher 就是在这个时候存储进去的。

```java
static {
    //省略一些代码
    registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
        new CachedServiceFetcher<LayoutInflater>() {
    @Override
    public LayoutInflater createService(ContextImpl ctx) {
        return new PhoneLayoutInflater(ctx.getOuterContext());
    }});
    //省略一些代码
}
```



### 为什么要放在 CachedServiceFetcher 中而不是直接创建呢？

ServiceFetcher 是一个抽象类，系统使用的一个具体实现类为 CachedServiceFetcher。从名字来看，主要是为了实现懒加载，当首次需要使用才触发初始化，避免浪费资源。



CachedServiceFetcher#getService 方法的实现

```java
public final T getService(ContextImpl ctx) {
    final Object[] cache = ctx.mServiceCache;
    synchronized (cache) {
        // Fetch or create the service.
        Object service = cache[mCacheIndex];//先从 ContextImpl 的缓存列表中获取，如果没有缓存再调用 CachedServiceFetcher#createService 方法创建 PhoneLayoutInflater
        
        if (service == null) {
            try {
                service = createService(ctx);//创建对应的 service
                cache[mCacheIndex] = service;//存储到缓存数组中
            } catch (ServiceNotFoundException e) {
                onServiceNotFound(e);
            }
        }
        return (T)service;
    }
}
```





### 小结

通过 from 的方式获取 LayoutInflater，最终调用的是 SystemServiceRegistry 的 getService 方法，该方法会从 ContextImpl 中维护的缓存数组中获取服务，如果不存在，则调用 createService 方法创建一个 PhoneLayoutInflater。（注：LayoutInflater 是一个抽象类，系统创建的是它的子类—— `PhoneLayoutInflater`）





onCreateView 是 PhoneLayoutInflater 中最重要的方法。为什么说它重要，后面会提到。

```Java
/** Override onCreateView to instantiate names that correspond to the
    widgets known to the Widget factory. If we don't find a match,
    call through to our super class.
*/
//为系统内置的控件添加前缀。比如为 TextView 添加前缀，结果为 android.widget.TextView
@Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
    for (String prefix : sClassPrefixList) {
        try {
            View view = createView(name, prefix, attrs);
            if (view != null) {
                return view;
            }
        } catch (ClassNotFoundException e) {
            // In this case we want to let the base class take a crack
            // at it.
        }
    }

    return super.onCreateView(name, attrs);
}
```



## 三、布局创建过程

一般我们在渲染 ListView 或者 RecyclerView 中的列表项时，都会调用 `inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) ` 方法（有时也会使用两个参数的方法）。可以看到该方法内部会调用 `inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)`。

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    //代码省略
    //获取 xml 解析器
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        //存储传入的父视图
        View result = root;

        try {
            int type;
            // 变量查找根节点
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            final String name = parser.getName();
            if (TAG_MERGE.equals(name)) {
                //1. 解析 merge 标签
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // 2. 不是 merge 元素就直接解析布局中的视图
 // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        // 生成布局参数
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            //如果 attachToRoot 为 false，就给 temp 设置布局参数
                            temp.setLayoutParams(params);
                        }
                    }

                    // 解析 temp 视图下的所有子 View
                    rInflateChildren(parser, temp, attrs, true);

                    // 如果 root 不为空并且 attachToRoot 为 true，那么将 temp 添加到 父视图中
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    //如果 root 为空 或者 attachToRoot 为 false，返回 xml 中的 root，而不是父视图
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            }

        return result;
    }
}

```

以上的 inflate 方法主要有以下几步

1. 解析 xml 的根标签
2. 如果根标签是 merge，调用 rInflate 进行解析，rInflate 会将 merge 标签下的所有子 View **直接添加到根标签中**
3. 如果标签是普通元素，调用 createFromTag，生成相应的 view。
4. 调用 rInflate 解析 temp 根元素下的所有子 View，并且将这些子 View 都添加到 temp 下
5. 返回解析到的根视图。

我们先从解析单个元素的 `createViewFromTag` 方法看起。

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // Apply a theme wrapper, if allowed and one is specified.
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }


    try {
        View view;
        //用户可以通过设置 LayoutInlflater 的 factory 来自行解析 View，默认这些 Factory 都为空，可忽略这段
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    // 解析内置 View 控件
                    view = onCreateView(parent, name, attrs);
                } else {
                    // 解析自定义控件
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    }
    //代码省略
```

**onCreateView 方法和 createView 方法有何不同**？   
前面的介绍中，我们有提到 LayoutInlflater 创建的实际类型为 `PhoneLayoutInlflater` ，PhoneLayoutInlflater 覆写了 onCreateView 方法，该方法就是在 View 的标签名前添加一个 `"android.widget." 或 "android.webkit." 或 "android.app."`前缀。然后再传递给 createView 解析。

- 也就是说**内置 View 和自定义 View 最终都调用了 createView 进行解析**。

**为什么要这么设计呢**？  
这是为了方便开发者在 xml 文件中更方便使用系统内置的 View（只需要写 View 名称而不需要写完整的路径），如果是自定义控件或者第三方库，写完整路径。

createView 的具体实现如下

```java
//根据完整路径的类名通过反射机制构造 View 对象
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    //从缓存中获取构造函数
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        // 缓存中找不到构造函数
        if (constructor == null) {
            //如果前缀（prefix）不为空，构造完整路径，并且加载该类
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            //代码省略
            //从 class 对象中获取构造函数
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            //代码省略
            }
        }

        Object[] args = mConstructorArgs;
        args[1] = attrs;
        //通过反射构造 View
        final View view = constructor.newInstance(args);
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        return view;

    } 
    //省略各种 catch、finally 代码
}
```

createView 方法中，如果控件名有前缀就先构造 View 的完整路径，并且将该类加载到虚拟机中

- 然后获取该类的构造函数并缓存起来，再通过构造函数来创建该 View 的对象，
- 最后将 View 对象返回，这就是解析单个 View 的过程

```java
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
    //获取树的深度
    final int depth = parser.getDepth();
    int type;
    //逐个元素解析
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();
        
        if (TAG_REQUEST_FOCUS.equals(name)) {
            parseRequestFocus(parser, parent);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {//解析 include 标签
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {解析 merge 标签
            throw new InflateException("<merge /> must be the root element");
        } else {
            //根据元素名进行解析
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            //递归调用进行解析，即深度优先遍历
            rInflateChildren(parser, view, attrs, true);
            //将解析到的 View 添加到它的父视图中
            viewGroup.addView(view, params);
        }
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

rInflate() 通过深度优先遍历来构造视图树。每解析到一个 View 元素就会递归调用 rInflate，直到这条路径下的最后一个元素，
然后在回溯过来将每个 View 元素添加到它们的 parent 中。

- 通过 rInflate 的解析之后，整棵视图树就构建完毕。当调用了 Activity 的 onResume 之后，之前通过 setContentView 设置的内容就会出现在我们的视野中。



## 四、常见问题

### inflate 方法有多种重载，它们的区别在哪里？

三个参数的 infalte 方法的各个参数的含义是什么？

`LayoutInflater#inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) `

resource 参数为布局资源。是一个 xml 文件。



若 root 不为空时

- 如果 attachToRoot 为 true，则它是 xml 文件的根布局的父布局。（也就是说，渲染出来的布局被 add root 中。同时，inflate 方法的返回值为 root）
- 如果 attachToRoot 为 false，它只是为 xml 文件的根布局 提供一组 LayoutParams 值。同时 inflate 方法返回的 xml 文件的根布局。



两个参数的 inflate 方法，内部也是调用三个参数的 infalte 方法

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

当 root 不为空的时候，inflate(@LayoutRes int resource, @Nullable ViewGroup root) 方法默认将渲染后的布局 add 到 root 中。



还有另外两种重载，区别仅在于它们的第一个参数都是 XmlPullParser，root 和 attachToRoot 参数表现与上述一致。



### 小结

创建流程大致如下：

首先查找根标签

- 如果是 merge，调用 rInflate
- 否则，调用 `createViewFromTag`
  - 如果是系统内置控件（通过名称中是否含有「.」来判断），调用 `PhoneLayoutInflater.onCreateView()` 方法添加前缀，
    - 处理后将完整路径传给 `LayoutInflater.createView()` 方法，该方法内部会通过反射的方式创建出对应的 view 实例。
  - 否则，直接调用 `LayoutInflater.createView()` 进行解析。该方法内部会通过反射的方式创建出对应的 view 实例。

## 五、拓展用法

前面提到在布局创建过程中，会调用 createViewFromTag 方法。

createViewFromTag 方法中，有下一段代码。

```java
View view;
//用户可以通过设置 LayoutInlflater 的 factory 来自行解析 View，默认这些 Factory 都为空，可忽略这段
if (mFactory2 != null) {
    view = mFactory2.onCreateView(parent, name, context, attrs);//如果Factory2不为空，调用 mFactory2 创建 view
} else if (mFactory != null) {
    view = mFactory.onCreateView(name, context, attrs);
} else {
    view = null;
}
```

```java
public interface Factory2 extends Factory {
    /**
     * Version of {@link #onCreateView(String, Context, AttributeSet)}
     * that also supplies the parent that the view created view will be
     * placed in.
     *
     * @param parent The parent that the created view will be placed
     * in; <em>note that this may be null</em>.
     * @param name Tag name to be inflated.
     * @param context The context the view is being created in.
     * @param attrs Inflation attributes as specified in XML file.
     *
     * @return View Newly created view. Return null for the default
     *         behavior.
     */
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}
```

我们可以自己实现 Factory 中的  onCreateView 方法。系统在填充 view 前会回调该方法，可以返回什么样的 view 已经其中的属性。



### 1.全局替换字体属性

因为字体是 TextView 的一个属性，为了加一个属性，我们就没必要去全部的布局中进行更改，只需要上我们的 onCreateView 中，发现是 TextView，就去设置我们对应的字体。

```java
public static Typeface typeface;
@Override
protected void onCreate(Bundle savedInstanceState)
{
    if (typeface == null){
        typeface = Typeface.createFromAsset(getAssets(), "xxxx.ttf");
    }
    LayoutInflaterCompat.setFactory2(LayoutInflater.from(this), new LayoutInflater.Factory2() {
        @Override
        public View onCreateView(String name, Context context, AttributeSet attrs) {
            return null;
        }

        @Override
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
            AppCompatDelegate delegate = getDelegate();
            View view = delegate.createView(parent, name, context, attrs);

            if ( view!= null && (view instanceof TextView)){
                ((TextView) view).setTypeface(typeface);
            }
            return view;
        }

    });
    
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}
```

### 2.动态换肤功能

基本原理和上面的更换字体类似，我们可以对做了标记的 View 进行识别，然后在 onCreateView 遍历到它的时候，更改它的一些属性，比如背景色等，然后再交给系统去生成 View。

### 3.无需编写 shape、selector，直接在 xml 设置值

[无需自定义 View，彻底解放 shape，selector 吧](https://juejin.im/post/5b9682ebe51d450e543e3495)

文章中讲到我们如果要设置控件的角度等属性值，不需要再去写特定的 shape 或者 selector 文件，直接在 xml 中写入：

```
  <Button
        android:id="@+id/btn"
        android:layout_width="130dp"
        android:layout_height="36dp"
        android:layout_marginTop="5dp"
        android:gravity="center"
        android:padding="0dp"
        android:text="跳转到列表"
        android:textColor="#4F94CD"
        android:textSize="20sp"
        app:bl_corners_radius="20dp
        app:bl_gradient_angle="0"
        app:bl_gradient_endColor="#4F94CD"
        app:bl_gradient_startColor="#63B8FF"
        app:bl_shape="rectangle" />
```





其原理也是通过自定义 Factory，在`Factory2#onCreateView`方法里面，判断 attrs 的参数名字，比如发现名字是我们制定的 stroke_color 属性，就去通过代码手动帮他去设置这个值,我们来查看下它的 onCreateView 方法：

```java
@Override
public View onCreateView(String name, Context context, AttributeSet attrs) {
	//省略一些代码
    if (typedArray.getBoolean(R.styleable.background_ripple_enable, false) &&
        typedArray.hasValue(R.styleable.background_ripple_color)) {
        int color = typedArray.getColor(R.styleable.background_ripple_color, 0);
            if (android.os.Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                Drawable contentDrawable = (stateListDrawable == null ? drawable : stateListDrawable);
                RippleDrawable rippleDrawable = new RippleDrawable(ColorStateList.valueOf(color), contentDrawable, contentDrawable);
                view.setClickable(true);
                view.setBackground(rippleDrawable);
            } else {
                StateListDrawable tmpDrawable = new StateListDrawable();
                GradientDrawable unPressDrawable = DrawableFactory.getDrawable(typedArray);
                unPressDrawable.setColor(color);
                tmpDrawable.addState(new int[]{-android.R.attr.state_pressed}, drawable);
                tmpDrawable.addState(new int[]{android.R.attr.state_pressed}, unPressDrawable);
                view.setClickable(true);
                view.setBackground(tmpDrawable);
            }
        }
        return view;
}
```





## 六、参考资料与学习资源推荐

- 《Android 源码设计模式解析与实战》
- [Android 技能树 — LayoutInflater Factory 小结](https://www.jianshu.com/p/8d8ada21ab82)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！