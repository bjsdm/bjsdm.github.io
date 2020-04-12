---
layout: post
title: HandlerThread源码分析
author: 不近视的猫
date: 2020-04-12
categories: blog
tags: []
description: HandlerThread源码分析
---


HandlerThread 属于 Android Handler 机制的一部分，若还未了解 Handler 机制的话，建议先看看[Handler源码分析](https://blog.csdn.net/m0_46278918/article/details/105262993)。

## HandlerThread是什么
首先，我们来看看，假如我们需要在`handleMessage`方法中，启用线程处理数据该怎么做：

```
        Handler handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        Log.i("处理数据的线程", Thread.currentThread().getName());
                    }
                }).start();
            }
        };
        Log.i("Handler所在的线程：", Thread.currentThread().getName());
        handler.sendEmptyMessage(1);
```

结果输出：

```
I/Handler所在的线程：: main
I/处理数据的线程: Thread-17
```

但是，假如对于 Handler 机制比较了解的话，就会清楚，其实`handleMessage`方法的执行线程是`Looper`所在的线程，也就是说，在创建`Handler`的时候，假如传入其它线程的`Looper`，这时，`handleMessage`方法就在其它线程运行了。

```

    class TestThread extends Thread {

        private Looper looper;

        @Override
        public void run() {
            Looper.prepare();
            synchronized (this){
                looper = Looper.myLooper();
                notifyAll();
            }
            Looper.loop();
        }

        public Looper getLooper() {
            if (looper == null) {
                synchronized (this){
                    try {
                        wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
            return looper;
        }
    }
```

```
        TestThread testThread = new TestThread();
        testThread.start();

        Handler handler = new Handler(testThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                Log.i("处理数据的线程", Thread.currentThread().getName());
            }
        };
        Log.i("Handler所在的线程：", Thread.currentThread().getName());
        handler.sendEmptyMessage(1);
```

结果输出：

```
I/Handler所在的线程：: main
I/处理数据的线程: Thread-15
```

### TestThread分析
为什么`TestThread`写得这么复杂？

1、不能直接返回`Looper.myLooper()`

`Looper.myLooper()`返回的是当前线程中的`Looper`。若直接使用`TestThread`对象直接调用`getLooper()`返回`Looper.myLooper()`，那么返回的`Looper`为调用`TestThread`对象的线程的`Looper`，在本文例子中，则为返回主线程的`Looper`。

所以，需要在`run()`方法中初始化`Looper`，并将其值保存起来。

2、为什么要使用`wait()`和`notifyAll()`

因为`Looper`是在`run()`中进行初始化的，也就是说开启线程进行初始化，是异步的，在调用完`TestThread`的`start()`方法后，直接获取`Looper`有可能为null，所以需要使用
`wait()`和`notifyAll()`确保一定能够获取得到`Looper`对象。

嗯？是不是走远了？怎么这么久都还没讲到`HandlerThread`是什么。

其实，**`HandlerThread`其实就是`TestThread`，只不过`TestThread`是简化版，但是核心逻辑基本一致。其继承于线程，并且对于新线程的`Looper`进行初始化并管理**，所以，应该叫做`LooperThread`或许更加合适。

## HandlerThread的简单使用
```

        //初始化HandlerThread，传入的值为线程名字
        HandlerThread handlerThread = new HandlerThread("ThreadName");
        //记得一定要调用start()，其启动线程初始化Looper
        handlerThread.start();

        //初始化Handler并传入其它线程的Looper
        Handler handler = new Handler(handlerThread.getLooper()){

            @Override
            public void handleMessage(Message msg) {
                Log.i("处理数据的线程", Thread.currentThread().getName());
            }
        };

        Log.i("Handler所在的线程：", Thread.currentThread().getName());
        //发送消息
        handler.sendEmptyMessage(1);
        //不使用后，要对于Looper进行退出操作
        handlerThread.quit();
```

代码的注释已经写得够明白了，下面我们来看看输出结果：

```
 I/Handler所在的线程：: main
 I/处理数据的线程: ThreadName
```

## HandlerThread源码分析（源码只保留关键部分，并非全部源码）
同样的，我们还是从简单使用的步骤，一步步分析：

### HandlerThread初始化
```
HandlerThread handlerThread = new HandlerThread("ThreadName");
```

其源码：

```
    public HandlerThread(String name) {
        super(name);
        ···
    }
```

`super(name)`是调用父类`Thread`的构造方法：

```
    public Thread(String name) {
        init(null, null, name, 0);
    }
```

这个我们就不深究了，这属于`Thread`的方法，知道这是为了初始化`Thread`即可。

### HandlerThread start() 的调用
```
handlerThread.start();
```

因为`handlerThread`本质也是一个`Thread`，所以，我们重点看看`handlerThread`对于`run()`的重写：

```
    @Override
    public void run() {
        ···
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        ···
        onLooperPrepared();
        Looper.loop();
        ···
    }
    
    protected void onLooperPrepared() {
    }

    
```

其实就是对于`Looper`进行初始化，并且把初始化后的`Looper`对象存储到`mLooper`中，并且调用`onLooperPrepared()`。

我们可以看到，`onLooperPrepared()`里面没有任何代码，用于用户重写该方法，执行些`Looper`初始化后的一些操作。

### HandlerThread的Looper获取
```
handlerThread.getLooper()
```

```
    public Looper getLooper() {
		···
        // 避免获取Looper的时候，Looper还未初始化（因为Looper是启动线程进行初始化）
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
```

### HandlerThread的quit()

```
handlerThread.quit();
```

```
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

```

```
    public void quit() {
    	//mQueue为MessageQueue
        mQueue.quit(false);
    }
```

结束消息队列，进而结束线程。

所以，**使用完`HandlerThread`一定要记得调用`quit()`，否则线程将会一直存在！**











