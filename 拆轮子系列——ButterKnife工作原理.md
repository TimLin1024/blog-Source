---
title: 拆轮子系列——ButterKnife工作原理
date: 2017-10-07 10:07:55
tags: 
- 源码分析
- 框架原理
categories:
- 源码分析
- 框架原理
---

## 概要

本篇博客主要包括以下三个部分

-   日常使用 ButterKnife 的方式以及 ButterKnife 的 bind 方法
-   Xxx_ViewBinding 类是如何生成的？
-   注解处理器是怎么注册的？

其中的代码基于 ButterKnife 8.6.0

本文假设你已经对注解有所了解。不了解注解的同学可以先看看[这篇文章](http://a.codekk.com/detail/Android/Trinea/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E6%B3%A8%E8%A7%A3%20Annotation) ，我们主要关注编译期注解。

<!--more-->

## 运行期 ButterKnife#bind

我们通常都是通过以下方式使用 ButterKnife 的。将要绑定的控件加上 @BindView 注解，在 onCreate 方法    完成 `setContentView(R.layout.activity_main);` 之后。调用 `ButterKnife.bind(this);` 对相应的  View 进行初始化。

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    @BindView(R.id.btn)
    Button mBtn;
    @BindView(R.id.radar_view)
    RadarView mRadarView;
  
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);//ButterKnife 的入口
    }
}
```

`public static Unbinder bind(@NonNull Activity target) ` ，在 Activity 中以该方法作为入口，首先获取 与 Activity 相关联的 window，然后再通过 window 获取 DecorView。最后通过 `createBinding` 方法创建相关绑定。

```java
public static Unbinder bind(@NonNull Activity target) {
  View sourceView = target.getWindow().getDecorView();
  return createBinding(target, sourceView);
}
```



```java
private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
  Class<?> targetClass = target.getClass();
  if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
  Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);//查找目标类的构造器

  if (constructor == null) {
    return Unbinder.EMPTY;
  }

  try {
    return constructor.newInstance(target, source);//利用反射创建一个实例，这里实际上调用了 Xxx_ViewBinding 的构造方法（由此可以看到，Butterknife 也是有用到一点反射的）
  } 
  //代码省略
}
```

```java
static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();//LinkedHashMap 作为缓存容器

@Nullable @CheckResult @UiThread
private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
  Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);//先从缓存中查找
  if (bindingCtor != null) {
    return bindingCtor;//缓存命中直接返回
  }
  String clsName = cls.getName();//获取类型
  //如果是 Framework 层的类，则直接返回 null
  if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
    return null;
  }
  try {
    Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");//加载类
    //noinspection unchecked
    bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);//获取构造器
  } catch (ClassNotFoundException e) {
    bindingCtor = findBindingConstructorForClass(cls.getSuperclass());//递归查找父类的构造器
  } catch (NoSuchMethodException e) {
    throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
  }
  BINDINGS.put(cls, bindingCtor);//加入到缓存中
  return bindingCtor;//返回构造器
}
```

以下为编译器生成的 MainActivity_ViewBinding 类，该类在构造方法中对那些在 MainActivity 中打了注解（例如：`@BindView`）的 View 进行了初始化。

```java
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;//目标类

  private View view2131427444;

  private View view2131427442;

  private View view2131427443;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(final MainActivity target, View source) {
    this.target = target;

    View view;
    view = Utils.findRequiredView(source, R.id.btn, "field 'mBtn' and method 'onViewClicked'");//findViewbyId
    target.mBtn = Utils.castView(view, R.id.btn, "field 'mBtn'", Button.class);//强制类型转换
    view2131427444 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {//注册监听器
      @Override
      public void doClick(View p0) {
        target.onViewClicked(p0);
      }
    });
    
    target.mRadarView = Utils.findRequiredViewAsType(source, R.id.radar_view, "field 'mRadarView'", RadarView.class);//findViewbyId
  }
```

这里插一个小话题：通常我们都会把不需要被外部使用到的成员变量声明为 private 的，但是回头看看我们使用 `@BindView`的 View 访问权限均为 defalut 。为什么呢？从 MainActivity_ViewBinding 的构造方法中可以看到对 View 进行赋值时直接使用了 `target.mBtn、 target.mRadarView `，如果声明为 private ，那就需要通过反射才能访问到对应的 View，这样会有性能上的损失。

### 小结

在运行时以 `ButterKnife#bind` 方法作为入口，首先以目标 class 为 key 到缓存（一个 `LinkedHashMap`）中查找相应的 binding class，如果命中直接返回。如果没有缓存，则通过类加载器加载对应的 `Xxx_ViewBinding` 类，并将其加入到缓存中，然后返回。最后调用 Xxx_ViewBinding 类的构造器（通过反射）创建一个实例。Xxx_ViewBinding 类的构造器中完成了对 View 的绑定。比如  `findViewById，setOnClickListener`

## 编译期生成代码

### Q：前面提到的 MainActivity_ViewBinding  类是如何生成的呢？

 Answer: 在编译时对类中的 Annotation 进行解析，通过 JavaPoet 生成相应的代码。

`编译时 Annotation` 指 @Retention 为 CLASS 的 Annotation，由编译器自动解析。开发人员需要做的
a. 自定义类继承自 AbstractProcessor
b. 重写其中的 process 函数
c. 注册注解处理器。注册完成后，**编译器在编译时自动查找所有继承自 AbstractProcessor 的类，然后调用他们的 process 方法去处理**

### 自定义 Annotation  用于存储元数据

以 BindView 注解为例

```java
@Retention(CLASS) @Target(FIELD)
public @interface BindView {
  /** View ID to which the field will be bound. */
  @IdRes int value();
}
```

-   `@Retention(CLASS)` ：表示保留到编译期
-   `@Target(FIELD)`：表示作用域为成员变量



### 注解处理器 ButterKnifeProcessor

ButterKnifeProcessor 继承自 AbstractProcessor ，AbstractProcessor 的主要源码如下，我们主要关注四个方法

```java
public abstract class AbstractProcessor implements Processor {
    protected ProcessingEnvironment processingEnv;
    private boolean initialized = false;

    protected AbstractProcessor() {//默认构造器
    }
	//获取支持的注解类型
    public Set<String> getSupportedAnnotationTypes() {
        SupportedAnnotationTypes var1 = (SupportedAnnotationTypes)this.getClass().getAnnotation(SupportedAnnotationTypes.class);//获取支持的注解类型
        if(var1 == null) {
            if(this.isInitialized()) {//如果已经初始化了，打印提醒
                this.processingEnv.getMessager().printMessage(Kind.WARNING, "No SupportedAnnotationTypes annotation found on " + this.getClass().getName() + ", returning an empty set.");
            }

            return Collections.emptySet();//返回空集
        } else {
            return arrayToSet(var1.value());//返回支持的注解集合
        }
    }

    public SourceVersion getSupportedSourceVersion() {//获取支持的源码版本
        SupportedSourceVersion var1 = (SupportedSourceVersion)this.getClass().getAnnotation(SupportedSourceVersion.class);//支持的源码版本
        SourceVersion var2 = null;
        if(var1 == null) {
            var2 = SourceVersion.RELEASE_6;
            if(this.isInitialized()) {
                this.processingEnv.getMessager().printMessage(Kind.WARNING, "No SupportedSourceVersion annotation found on " + this.getClass().getName() + ", returning " + var2 + ".");
            }
        } else {
            var2 = var1.value();
        }

        return var2;
    }

    public synchronized void init(ProcessingEnvironment var1) {
        if(this.initialized) {
            throw new IllegalStateException("Cannot call init more than once.");
        } else {
            Objects.requireNonNull(var1, "Tool provided null ProcessingEnvironment");
            this.processingEnv = var1;
            this.initialized = true;
        }
    }

    public abstract boolean process(Set<? extends TypeElement> var1, RoundEnvironment var2);
}
```



-   `init(ProcessingEnvironment env)`：每个注解处理器都必须有个空的构造方法。不过，有一个特殊的 init 方法，它会被注解处理器工具传入一个 ProcessingEnvironment 作为参数来调用。ProcessingEnvironment 提供了一些有用的工具类，如 Elements，Types 和 Filter。我们后面会用到它们。
-   `process(Set<? extends TypeElement> annotations, RoundEnvironment env)`：这个方法可以看做每个处理器的 main 方法。你要在这里写下你的扫描，判断和处理注解的代码，并生成 java 文件。通过传入的 RoundEnvironment 参数，你可以**查询使用了某个特定注解的所有元素**，我们稍后会看到。
-   `getSupportedAnnotationTypes( )`：这里你需要说明这个处理器需要针对哪些注解来注册。注意返回类型是一个字符串的 Set，包含了你要用这个处理器处理的注解类型的全名
-   `getSupportedSourceVersion( )`：用于指定你使用的 java 版本。通常会选择返回`SourceVersion.latestSupported( )`。当然，你也可以指定具体 java 版本：比如`return SourceVersion.RELEASE_7`;

ButterKnife 的解析工作就是通过 `ButterKnife#process` 方法实现的。

```java
private Filer filer;//成员变量

@Override
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);//扫描并解析目标类，获取 存储绑定信息的 bindingMap。
  	//遍历获取到的注解
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingSet binding = entry.getValue();

      JavaFile javaFile = binding.brewJava(sdk);//使用 javaPoet 生成 java 文件
      try {
        javaFile.writeTo(filer);//调用 javaPoet 的方法，将内容写入文件
      } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
      }
    }

    return false;
  }
}
```

process 方法首先调用 findAndParseTargets 方法获取解析的目标类

```java
 private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

    scanForRClasses(env);

    // 处理 @BindArray 元素.
    for (Element element : env.getElementsAnnotatedWith(BindArray.class)) {
      if (!SuperficialValidation.validateElement(element)) continue;
      try {
        parseResourceArray(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindArray.class, e);
      }
    }

	//代码省略
   
    // 遍历处理每一个打了 @BindView 注解的元素
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      // 解析 BindView 注解
      try {
        parseBindView(element, builderMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }

    // 处理每一个关联有监听器的注解
    for (Class<? extends Annotation> listener : LISTENERS) {
      findAndParseListener(env, listener, builderMap, erasedTargetNames);
    }

    // Associate superclass binders with their subclass binders. This is a queue-based tree walk
    // which starts at the roots (superclasses) and walks to the leafs (subclasses).
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
      Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

      TypeElement type = entry.getKey();
      BindingSet.Builder builder = entry.getValue();

      TypeElement parentType = findParentType(type, erasedTargetNames);
      if (parentType == null) {
        bindingMap.put(type, builder.build());
      } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          // Has a superclass binding but we haven't built it yet. Re-enqueue for later.
          entries.addLast(entry);
        }
      }
    }

    return bindingMap;//返回 bindingMap，用于后续 javapoet 生成代码
  }
```

上面的方法是以注解为单位，处理各个使用了注解的属性，然后将它们存放到一个 Map 中。（该 Map 以属性所在类为 key，以属性集 BindingSet.Builder  为 value）。那么具体是如何解析的呢？

对于 ButterKnife 我们最常用的是 `@BindView`，针对 BindView 我们主要看 `parseBindView` 方法，该方法将使用了`@BindView` 的属性的绑定信息，存放到以该属性所在类为 key ，值为  `BindingSet.Builder` 的Map 中，供后续 javaPoet 生成代码使用。

```java
 private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
      Set<TypeElement> erasedTargetNames) {
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();//获取该元素所在的类对应的元素。

    // 检查常见的代码生成限制
    boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
        || isBindingInWrongPackage(BindView.class, element);

    // 检查目标类型是否继承自 View
    TypeMirror elementType = element.asType();
    if (elementType.getKind() == TypeKind.TYPEVAR) {
      TypeVariable typeVariable = (TypeVariable) elementType;
      elementType = typeVariable.getUpperBound();
    }
    Name qualifiedName = enclosingElement.getQualifiedName();//获取全称
    Name simpleName = element.getSimpleName();//获取简单名
	//代码省略……
	//如果不是接口而且不是 View 的子类，则打印错误信息，将 hasError 置为true
    if (hasError) {
      return;
    }

    // 下面开始拼接要绑定的属性的信息
   
    int id = element.getAnnotation(BindView.class).value();//获取 R.id.xx

    BindingSet.Builder builder = builderMap.get(enclosingElement);//获取该属性所在的类的 builder
    QualifiedId qualifiedId = elementToQualifiedId(element, id);//从绑定记录中获取，如果相应的 id 已经被绑定过了，则报错，并打印相应的错误信息
    if (builder != null) {
      String existingBindingName = builder.findExistingBindingName(getId(qualifiedId));
      if (existingBindingName != null) {
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName,
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      //创建一个 BindingBuilder
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
    }

    String name = simpleName.toString();//简单类名
    TypeName type = TypeName.get(elementType);//获取该属性所在类的类名
    boolean required = isFieldRequired(element);

    builder.addField(getId(qualifiedId), new FieldViewBinding(name, type, required));//创建 Field 并添加到 BindingSet.Builder 的 field 中

    // Add the type-erased version to the valid binding targets set.
    erasedTargetNames.add(enclosingElement);
  }
```

假设我们在某一个类中重复绑定了相同的 id，比如下面这个例子对 R.id.app_bar_layout 进行了两次绑定：

```java
@BindView(R.id.app_bar_layout)
AppBarLayout mAppBarLayout;
@BindView(R.id.app_bar_layout)//绑定了相同的 id，会报错
AppBarLayout mAppBarLayout2;
```

在编译期会报如下错误：

`Error:(36, 18) 错误: Attempt to use @BindView for an already bound ID 2131230757 on 'mAppBarLayout'. (com.android.rdc.librarysystem.MainActivity.mAppBarLayout2)`

#### 小结

ButterKnife 是以这样的方式来处理注解的：ButterKnife 支持多种注解（比如 @BindView 、@BindColor），这些注解被按照一定的顺序处理。获取使用了某个注解的所有 Element，对该 Element 的进行处理，生成相应的 FieldXxxBinding 然后将它添加到该 Element 所在类的 `BindingSet.Builder` 中。

也就是说，ButterKnife 是以注解为单位进行解析，处理完了，就把它放到相应属性所在类对应的 Set 里面（而不是一次扫描一个类，然后对该类中属性使用到的注解进行处理，一次性生成某个类的所对应的 `BindingSet.Builder` ）。



#### 通过 JavaPoet 生成 java 文件

正如其名，poet 指诗人，也就是作诗的人。java poet 指的是能够自动写 java 源代码的库。

javapoet 里面常用的几个类：

-   `TypeName` Java 系统的所有类型，加上 void 类型
-   `MethodSpec` 代表一个构造函数或方法声明。
-   `TypeSpec` 代表一个类，接口，或者枚举声明。
-   `FieldSpec` 代表一个成员变量，一个字段声明。
-   `JavaFile`包含一个顶级类的 Java 文件。

回到前面的 process 方法，可以看到 `JavaFile javaFile = binding.brewJava(sdk);` `javaFile.writeTo(filer);`

BindingSet#brewJava 。该方法通过 Javapoet 生成一个 java 文件

```java
JavaFile brewJava(int sdk) {
  return JavaFile.builder(bindingClassName.packageName(), createType(sdk))
      .addFileComment("Generated code from Butter Knife. Do not modify!")//添加文件注释
      .build();
}
```

关于 javapoet 的介绍详见[这篇文章](http://www.jianshu.com/p/95f12f72f69a)

### 注解处理器的注册

#### 方法 1：使用 google 提供的注册处理器库

最简单的方式是使用 google 提供的一个注册处理器的库。 `compile 'com.google.auto.service:auto-service:1.0-rc2'`

然后`@AutoService(Processor.class) `:向 javac 注册我们这个自定义的注解处理器，这样，在 javac 编译时，才会调用到我们这个自定义的注解处理器方法。

AutoService 这里主要是用来生成
`META-INF/services/javax.annotation.processing.Processor`文件的。

#### 方法 2：手动注册

如果不使用上述处理库，那么，你需要自己进行手动配置进行注册，具体手动注册方法如下：
1.创建一个
`META-INF/services/javax.annotation.processing.Processor` 文件，
其内容是一系列的自定义注解处理器完整有效类名集合，以换行切割：

```
com.example.MyProcessor
com.foo.OtherProcessor
net.blabla.SpecialProcessor
```

2.将自定义注解处理器和
META-INF/services/javax.annotation.processing.Processor 打包成一个.jar 文件。所以其目录结构大概如下所示：

```
MyProcessor.jar
  - com
      - example
          - MyProcessor.class

  - META-INF
      - services
          - javax.annotation.processing.Processor
```



ButterKnife 使用的是第一种方式。

```java
@AutoService(Processor.class)
public final class ButterKnifeProcessor extends AbstractProcessor {
  //...
}
```



## 总结

ButterKnife 在**编译的时候**帮我们**自动生成**了绑定的代码，然后在**运行的时候调用**就行了。

-   首先我们在需要绑定 View 的 地方使用 @BindView 或者 ButterKnife 提供的其他注解，比如 @BindColor 
-   在编译期  ButterKnifeProcessor 的 process 函数会被调用，在其中获取使用了相应注解的相关方法/成员变量信息，通过 javapoet 生成 Xx_ViewBinder 和 Xx_ViewBinding 类。
-   运行时，onCreate 方法中通常需要 `ButterKnife.Bind()`方法，从此处进入，通过 反射调用 Xx_ViewBinding 的构造方法对 View 进行初始化。

## 参考资料与学习资源推荐

-   [公共技术点之 Java 注解 Annotation](http://a.codekk.com/detail/Android/Trinea/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E6%B3%A8%E8%A7%A3%20Annotation)
-   [ButterKnife 源码分析](http://www.jianshu.com/p/0f3f4f7ca505)
-   [深入理解 ButterKnife，让你的程序学会写代码](http://dev.qq.com/topic/578753c0c9da73584b025875#rd)
-   [javapoet——让你从重复无聊的代码中解放出来](http://www.jianshu.com/p/95f12f72f69a)



由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问欢迎在下面评论区告诉我，请对问题描述尽量详细，以帮助我可以快速找到问题根源。谢谢！

