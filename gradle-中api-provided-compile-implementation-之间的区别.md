---
title: gradle 中 api provided compile implementation 之间的区别
date: 2018-06-24 14:22:13
tags: 
- Gradle
categories:
- Gradle

---



## 概述

`api、provided、compile、implementation` 都是 gradle 添加依赖时的选项，不同的选项表示不同的依赖关系。

其中 api、implementation 是 Gradle 3.4 引入的新的依赖配置，用来代替 `compile` 依赖配置。其中 `api` 和以前的 `compile` 依赖配置是一样的。使用 api/compile 的依赖方式，会向外界暴露本模块所依赖的库的接口，使用 implementation 则不会。

<!--more-->

## 解决了什么问题？

为什么要做这样的修改呢？使用 `implementation` 依赖配置，可以显著减少构建的时间。

考虑这样的场景，有如下六个模块：

![](https://raw.githubusercontent.com/TimLin-pro/Graph/master/markdown/gradle%20%E6%A8%A1%E5%9D%97%E4%BE%9D%E8%B5%96%E7%A4%BA%E4%BE%8B.jpg)


D、E、F 三个模块都依赖了 Lib 库。

假设 D、E、F 三个模块都以 `implementation` 的方式引用 Lib 库，那么上层的 A，B，C 三个模块**在编译期**都无法引用到 Lib 库中的内容。因此当 Lib 发生变化的时候，只有 D E F 这三个模块需要重新编译，A,B,C 则不需要（因为 DEF 三个模块对外的接口都没有改变）。

假设 D、E、F 三个模块都以 `compile/api`  的方式引用 Lib 库（对外暴露了自己引用的库的接口），那么无论上层的 A，B，C 三个模块对 D、E、F 模块的依赖是 compile、api 方式还是 implementation 方式，它们（A、B、C）都可以引用到 Lib 库中的内容。这种情况下当 Lib 发生变化的时候，A B C D E F 这三个模块都需要重新编译。

Public 可以

## 其它变化

在该版本中，官方团队也把之前命名的不规范的地方修改了，provided 改成了 compileOnly，apk 改为了 runtimeOnly。改变的只是名称，功能没变。



### compileOnly 的使用场景

包括但是不限于下面的两种：

1. 编译时所需的依赖性，但在运行时从不需要，例如纯源注释或注释处理器;
2. API 在编译时需要，但其实现由消费库，应用程序或运行时环境提供的依赖项。

### runtimeOnly 的使用场景

依赖项仅在运行时对模块及其消费者可用。



## 小结

| 新配置           | 已弃用配置 | 行为                                                         |
| ---------------- | ---------- | ------------------------------------------------------------ |
| `implementation` | `compile`  | 依赖项在编译时对模块可用，并且仅在运行时对模块的消费者可用。 对于大型多项目构建，使用 `implementation` 而不是 `api`/`compile` 可以**显著缩短构建时间**，因为它可以减少构建系统需要重新编译的项目量。 大多数应用和测试模块都应使用此配置。 |
| `api`            | `compile`  | 依赖项在编译时对模块可用，并且在编译时和运行时还对模块的消费者可用。 此配置的行为类似于 `compile`（现在已弃用），一般情况下，您应当仅在库模块中使用它。 应用模块应使用 `implementation`，除非您想要将其 API 公开给单独的测试模块。 |
| `compileOnly`    | `provided` | 依赖项仅在编译时对模块可用，并且在编译或运行时对其消费者不可用。 此配置的行为类似于 `provided`（现在已弃用）。 |
| `runtimeOnly`    | `apk`      | 依赖项仅在运行时对模块及其消费者可用。 此配置的行为类似于 `apk`（现在已弃用）。 |



>  **注**：`compile`、`provided` 和 `apk` 目前仍然可用。 不过，它们将在下一个主要版本的 Android 插件中消失。

日常开发中一般都是用 implementation，部分情况下会用到 api，其他的依赖方式一般不怎么用到。记住 compile、api 依赖方式是「传递性的」，而 implementation 依赖方式的是「非传递性」的。

## 参考资料与学习资源推荐

- [迁移到 Android Plugin for Gradle 3.0.0](https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration)

- [Implementation vs API dependency](https://jeroenmols.com/blog/2017/06/14/androidstudio3/)

- [Android Studio3.x 新的依赖方式（implementation、api、compileOnly）](https://blog.csdn.net/yuzhiqiang_1993/article/details/78366985)

   

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！