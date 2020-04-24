---
layout: post
title: IntentService源码分析
author: 不近视的猫
date: 2020-04-20
categories: blog
tags: []
description: IntentService源码分析
---


`Service`作为 Android 四大组件之一，其与`Activity`的区别可以理解为没有界面的`Activity`也就是说，`Service`要做的事情更多是置于后台进行操控，就像幕后黑手一样在无声无息中制造大量事件，因此，较为耗时的操作以及需要长时间使用但是却不需要展现界面的操作，都置于`Service`中运行，例如：数据处理以及`Socket`连接等。

而`IntentService`继承于`Service`，可以理解为对于`Service`进行一层封装，但是，封装了什么？

估计大部分人都知道，`IntentService`与`Service`的区别重点在于`IntentService`内部会启动一个线程进行操作执行，并且在执行完操作后，会自动停止该服务，而`Service`需要自己启动线程并且自己管理什么时候进行销毁，所以不注意的话，会导致应该被销毁的`Service`一直在后台存活，从而导致内存浪费。

## IntentService的简单使用
MyService 代码：

```
public class MyService extends IntentService {

    private int count = 0;

    public MyService() {
        super("MyService");
    }


    @Override
    protected void onHandleIntent(Intent intent) {
        //count每次+1，并输出当前的线程名字和count值
        count++;
        Log.i(Thread.currentThread().getName(), "count = " + count);
    }
}
```

在`AndroidManifest`中注册：

```
<service android:name=".MyService" />
```

循环启动`MyService`

```
        for (int i = 0; i < 5; i++) {
            Intent intent = new Intent(this, MyService.class);
            startService(intent);
        }
```

结果输出：

```
I/IntentService[MyService]: count = 1
I/IntentService[MyService]: count = 2
I/IntentService[MyService]: count = 3
I/IntentService[MyService]: count = 4
I/IntentService[MyService]: count = 5
```

## IntentService源码分析（源码只保留关键部分，并非全部源码）

既然`IntentService`集成于`Service`，那么，我们就从`Service`的生命周期开始看起：

### 构造方法

```
    public IntentService(String name) {
        super();
        mName = name;
    }
```

设置`name`属性

### onCreate()

```
    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```

- 创建`HandlerThread`，并把构造方法中的`name`作为基础进行拼接为`IntentService[" + mName + "]`作为线程的名字，详情可以看看[HandlerThread源码解析](https://blog.csdn.net/m0_46278918/article/details/105469007)
- 以`HandlerThread `的`Looper`作为参数，创建`ServiceHandler`

我们来看看`ServiceHandler`

```
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

`handleMessage`里面只有两个操作：

- 调用`onHandleIntent`方法
- 将当前`Service`进行销毁

其实这两个方法就是`IntentService`和`Service`的区别：

- 在新线程中执行`onHandleIntent`方法
- 执行完`onHandleIntent`后进行该`Service`的销毁

不过，我们要注意，**这里销毁`Service`是使用`stopSelf(int startId)`，而不是`stopSelf()`**。使用`stopSelf(int startId)`，假如还有其它`startId`存在的话，其`Service`不会被销毁，若使用`stopSelf()`，则会把当前`Service`直接销毁。

### onStart(@Nullable Intent intent, int startId)
```
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        //这里的startId，对应stopSelf(int startId)中的startId
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```

创建`Message`，并使用`ServiceHandler`发送到消息队列中。

也就是会执行`ServiceHandler`的`handleMessage`方法。

至此，`IntentService`的大致流程已经结束，不过还有一点，就是`Looper`是开启循环遍历消息队列的，所以在`Service`结束的时候，要记得把该队列进行退出：

```
    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }
```
