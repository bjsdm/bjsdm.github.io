---
layout: post
title: 重拾线程——FutureTask 是如何吃掉异常（2）
author: 不近视的猫
date: 2022-04-25
categories: blog
tags: []
description: 重拾线程——FutureTask 是如何吃掉异常（2）
---


## 前言

首先，我们先写个示例代码来运行 FutureTask：

```
    val futureTask = FutureTask {
        println("futureTask 正在执行")
        throw Exception("我是测试异常")
        println("futureTask 发生异常了！")
        return@FutureTask "1"
    }
    Thread(futureTask).start()
```

运行结果：

```
futureTask 正在执行

Process finished with exit code 0
```

很明显，没有抛出异常。但是假如调用了 `get()` ：

```
    futureTask.get()
```

运行结果：

```
futureTask 正在执行
Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.Exception: 我是测试异常
	at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
	at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
	at com.bjsdm.testkotlin.KotlinTestKt.main(KotlinTest.kt:17)
Caused by: java.lang.Exception: 我是测试异常
	at com.bjsdm.testkotlin.KotlinTestKt.main$lambda-0(KotlinTest.kt:11)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.lang.Thread.run(Thread.java:829)

Process finished with exit code 1
```

下面，我们就来分析下，FutureTask 是如何吃掉异常，而调用 `get()` 的时候，为什么会抛出异常。

## FutureTask 是如何吃掉异常

首先，我们看下：

```
Thread(futureTask)
```

其构造方法为：

```
    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
```

最终会调用到这里：

```
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        ...
        this.target = target;
        ...
    }
```

主要就是将 `Runnable target` 存储起来，并且进行其它一些初始化和拦截的操作。

然后，我们需要看下 `Thread(futureTask).start()` 的 `start()` 方法：

```
    public synchronized void start() {
        ...
        try {
            nativeCreate(this, stackSize, daemon);
            ...
        } finally {
            ...
        }
    }
```

```
    private native static void nativeCreate(Thread t, long stackSize, boolean daemon);
```

这里是使用 native 方法去开启线程，所以，我们需要转换思路。我们都知道，在线程执行的时候，都会调用 `run()`，所以，我们来看看这：

```
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

由于 target 为 FutureTask，所以，我们看看 FutureTask 的 `run()` 是怎么运行的：

```
    public void run() {
        ...
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                ...
            }
        } finally {
            ...
        }
    }
```

主要逻辑为：

- 获取当前的 callable 示例
- 执行 callable 的 `call()`
- 对于 `call()` 的执行进行了 try catch 操作，当 `call()` 发生异常的时候，会调用 `setException(ex)`

首先，我们来看看 callable 是怎么来的？

```
	private Callable<V> callable;

    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

所以，这个 callable 其实就是我们创建 FutureTask 传入的 匿名内部类：

```
	FutureTask {
        println("futureTask 正在执行")
        throw Exception("我是测试异常")
        println("futureTask 发生异常了！")
        return@FutureTask "1"
    }
```

所以，这就能理解为什么 FutureTask 在运行时出现异常，没有崩溃日志输出的原因。

剩下，我们就需要了解下，出现异常后，`setException(ex)` 干了什么：

```
    protected void setException(Throwable t) {
        ...
        outcome = t;
        U.putOrderedInt(this, STATE, EXCEPTIONAL);
        ...
    }
```

- 将异常存储到 outcome 中
- 将状态设置为 EXCEPTIONAL


## get() 为什么会抛出异常

我们先来看看 `get()` 做了什么：

```
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```

线程的状态标识：

```
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

所以，在异常的情况下，是直接运行 `report(s)` ：

```
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```

`throw new ExecutionException((Throwable)x);`，就是把 outcome 存储的异常抛出来了，所以，这就能理解为什么 `get()` 会抛出异常。

































