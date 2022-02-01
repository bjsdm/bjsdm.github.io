---
layout: post
title: sleep()为什么要 try catch
author: 不近视的猫
date: 2022-02-02
categories: blog
tags: []
description: sleep()为什么要 try catch
---

## 前言

当我们在 Java 中使用 sleep() 让线程休眠的时候，总是需要使用 try catch 去包含它：

```
        try {
            sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

但是，我们却很少在 catch 中执行其它代码，仿佛这个 try catch 是理所当然一样，但其实，存在即合理，不可能无缘无故地多出一个 try catch。

在给出原因之前，我先要讲讲另外一个知识点，那就是如何去停止线程。

## 如何停止线程
### stop()

最直接了断的方法就是调用 stop()，它能直接结束线程的执行，就如：

```
        // 开启线程循环输出当前的时间
        Thread thread = new Thread(){
            @Override
            public void run() {
                while (true){
                    System.out.println(System.currentTimeMillis());
                }
            }
        };
        thread.start();
        // 睡眠两秒后停止线程
        try {
            sleep(2000);
            thread.stop();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

输出结果：

```
···
1643730391073
1643730391073
1643730391073
1643730391073
1643730391073

Process finished with exit code 0
```

很明显是能够把线程暂停掉的。但是，该方法现在被标记遗弃状态。

<img src="https://img-blog.csdnimg.cn/f794ac2e77d7484ebe30363d89fcbd71.png" width = "450" >

大概意思就是：

这个方法原本是设计用来停止线程并抛出线程死亡的异常，但是实际上它是不安全的，因为它是直接终止线程解放锁，这很难正确地抛出线程死亡的异常进行处理，所以，更好的方式是设置一个判断条件，然后在线程执行的过程中去判断该条件以去决定是否要进行停止线程，这样就能进行处理并有序地退出这个线程。假如一个线程等待很久的话，那才会直接中断线程并抛出异常。

按照上面所说，我们可以设置一个变量来进行控制，当然，我们可以声明一个 bool 类型进行判断，但是更好的方式是使用 interrupt()。

### interrupt()
源码：

```
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
```

`Just to set the interrupt flag` 这里很明显就是说明只是进行了一个中断标记而已，并不会直接中断线程，所以，需要使用 interrupt() 的时候，我们还要在线程中进行 interrupt 的状态判断：

```

        // 开启线程循环输出当前的时间
        // 每次循环前都去判断该线程是否被中断了
        Thread thread = new Thread(){
            @Override
            public void run() {
                while (true){
                    if(isInterrupted()){
                        return;
                    }
                    System.out.println(System.currentTimeMillis());
                }
            }
        };
        thread.start();
        // 睡眠两秒后标记中断线程
        try {
            sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

输出结果：

```
···
1643731839919
1643731839919
1643731839919

Process finished with exit code 0
```

这样也能正常的中断线程。

<img src="https://img-blog.csdnimg.cn/20210326220150376.jpg" width = "150" >

好了，如何中断线程到这里讲完了，大家拜拜~~

喂！等下，这文章是不是还有东西没讲 (#`O′)，sleep()为什么要 try catch 还没说。

<img src="https://img-blog.csdnimg.cn/2021042310500417.png" width = "150" >

好像是这样。

## 中断等待

我们再看看 sleep() 被标记为遗弃的说明：

```
 If the target thread waits for long periods (on a condition variable, for example),
 the interrupt method should be used to interrupt the wait.
```

也就是说，当线程在等待过久的时候，interrupt()  应该去中断这个等待。

所以，原因就找到了，要加 try catch，是因为当线程在 sleep() 的时候，调用线程的 interrupt() 方法，就会直接中断线程的 sleep 状态，抛出 InterruptedException。

因为调用 interrupt() 的时候，其实就是想尽快地结束线程，所以，继续的 sleep 是没有意义的，应该尽快结束。

```
        // 开启线程
        // 线程睡眠 10 秒
        Thread thread = new Thread(){
            @Override
            public void run() {
                try {
                    sleep(10000);
                } catch (InterruptedException e) {
                    System.out.println("sleep 状态被中断了！");
                    e.printStackTrace();
                }
            }
        };
        thread.start();
        // 睡眠两秒后标记中断线程
        try {
            sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

输出结果：

```
sleep 状态被中断了！
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at com.magic.vstyle.TestMain$1.run(TestMain.java:16)

Process finished with exit code 0
```

这时，我们就可以 catch 到这个异常后进行额外操作，例如回收资源等。这时，停止线程就是一种可控的行为了。






