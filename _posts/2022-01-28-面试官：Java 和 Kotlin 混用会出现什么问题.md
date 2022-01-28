---
layout: post
title: 面试官：Java 和 Kotlin 混用会出现什么问题
author: 不近视的猫
date: 2022-01-28
categories: blog
tags: []
description: 面试官：Java 和 Kotlin 混用会出现什么问题
---
## 前言

这其实是上年面试时遇到的问题，后续去搜索，都没找到合适的答案，直至在工作中真的写到这 bug 后，才知道，Java 和 Kotlin 的混用，还是真的有坑的，真是血与泪的教训！

<img src="https://img-blog.csdnimg.cn/20210326215055408.jpg" width = "150" >

## 原由

我们都知道，在纯 Java 开发中，很容易出现 `NullPointerException`，而 Kotlin 的空安全就能很大一个程度避免该问题，也就是在声明类型的时候，就决定好该类型是否能够容纳 null，以此，我们来写个 kotlin 方法：

`KotlinUtils.kt`：

```
fun printMsg(msg : String){
    println("msg: $msg")
}
```

然后使用 Java 代码去调用它：

```
KotlinUtilsKt.printMsg("不近视的猫");
```

输出结果：

```
I/System.out: msg: 不近视的猫
```

em... 很好，没什么问题

<img src="https://img-blog.csdnimg.cn/20210326215055271.jpg" width = "100" >

但，假如我们使用 Java 传个 null 进去试试？

```
KotlinUtilsKt.printMsg(null);
```

运行：

```
Caused by: java.lang.NullPointerException: Parameter specified as non-null is null: method kotlin.jvm.internal.Intrinsics.checkNotNullParameter, parameter msg
        at com.bjsdm.testproject.KotlinUtilsKt.printMsg(Unknown Source:2)
        at com.bjsdm.testproject.JavaUtils.test(JavaUtils.java:5)
```

<img src="https://img-blog.csdnimg.cn/20210329222955824.png" width = "120" >

崩溃了。

但是，大家会觉得，这好像没什么，到时我记得传非 null 就好了。可惜事实没这么简单，因为我们很多都是通过 Gson 去直接解析服务器数据的，所以在直接使用这些字段的时候，很容易就忘了判空，因为在测试阶段很难发现这种情况，所以，往往出错的时候，都是在线上。

所以，若是提供给外部使用的方法，尽量参数设置为可空，后续做好判空。


---

## 猜你喜欢

- <a href="https://juejin.cn/post/6953263787621220360">面试官：如何提高Message的优先级</a>
- <a href="https://juejin.cn/post/6945499630276706311">制作一个永远不会崩溃的App</a>
- <a href="https://juejin.cn/post/6945380865404829703">自定义Gradle Plugin+字节码插桩</a>
- <a href="https://juejin.cn/post/6943829469106798629">从手写ButterKnife到掌握注解、AnnotationProcessor</a>
- <a href="https://juejin.cn/post/6942309139947356191">来，带你手写个性能检测工具</a>

创作不易，你的点赞是我最大的支持。

---

这是我的公众号，关注获取第一信息！！欢迎关注支持下，谢谢！

<img src="https://img-blog.csdnimg.cn/20210328021432830.png" width = "500" >
