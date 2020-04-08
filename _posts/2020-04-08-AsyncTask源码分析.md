---
layout: post
title: AsyncTask源码分析
author: 不近视的猫
date: 2020-04-08
categories: blog
tags: []
description: AsyncTask源码分析
---



AsyncTask，Android 实现异步方式之一，即可以在子线程进行数据操作，然后在主线程进行 UI 操作，Handler 也可以实现异步，详情可以看看[Handler源码分析](https://blog.csdn.net/m0_46278918/article/details/105262993)

## AsyncTask的简单使用
### 示例

同样的，我们先看看 AsyncTask 如何进行简单使用：

```
        AsyncTask<Boolean, Integer, String> asyncTask = new AsyncTask<Boolean, Integer, String>() {

            @Override
            protected void onPreExecute() {
                super.onPreExecute();
                Log.i("TAG", "onPreExecute：正在执行前的准备操作");
            }

            @Override
            protected String doInBackground(Boolean... booleans) {
                Log.i("TAG", "doInBackground：获得参数" + booleans[0]);
                for (int i = 0; i < 5; i++) {
                    publishProgress(i);
                }
                return "任务完成";
            }

            @Override
            protected void onProgressUpdate(Integer... values) {
                super.onProgressUpdate(values);
                Log.i("TAG", "onProgressUpdate：进度更新" + values[0]);
            }

            @Override
            protected void onPostExecute(String s) {
                super.onPostExecute(s);

                Log.i("TAG", "onPostExecute：" + s);
            }
        };
        Log.i("TAG", "开始调用execute");
        asyncTask.execute(true);
```

输出结果：

```
2020-04-02 17:14:42.029 4995-4995 I/TAG: 开始调用execute
2020-04-02 17:14:42.029 4995-4995 I/TAG: onPreExecute：正在执行前的准备操作
2020-04-02 17:14:42.030 4995-5118 I/TAG: doInBackground：获得参数true
2020-04-02 17:14:45.774 4995-4995 I/TAG: onProgressUpdate：进度更新0
2020-04-02 17:14:45.774 4995-4995 I/TAG: onProgressUpdate：进度更新1
2020-04-02 17:14:45.774 4995-4995 I/TAG: onProgressUpdate：进度更新2
2020-04-02 17:14:45.774 4995-4995 I/TAG: onProgressUpdate：进度更新3
2020-04-02 17:14:45.774 4995-4995 I/TAG: onProgressUpdate：进度更新4
2020-04-02 17:14:45.775 4995-4995 I/TAG: onPostExecute：任务完成
```

### 创建说明

首先，在创建 AsyncTask 的时候，需要传入三个泛型数据`AsyncTask<Params, Progress, Result>`，其分别对应着

```
   protected abstract Result doInBackground(Params... params);
```

```
    @MainThread
    protected void onProgressUpdate(Progress... values) {
    }
```

```
    @MainThread
    protected void onPostExecute(Result result) {
    }
```

从注解可以看出，`onProgressUpdate`和`onPostExecute`是在主线程执行。

### 执行流程

- AsyncTask 调用`execute(Params... params)`方法
- `onPreExecute() `被调用，该方法用于在执行后台操作前进行一些操作，例如：弹出个加载框等
- `doInBackground(Params... params) `被调用，该方法用于进行一些复杂的数据处理，例如数据库操作等
- 在`doInBackground`进行操作的过程中，可以通过`publishProgress(Progress... values)`进行进度更新，从而自动调用`onProgressUpdate(Progress... values) `
- 当`doInBackground`执行完毕后，返回数据，将会调用`onPostExecute(Result result)`

## 源码分析（源码只保留关键部分，并非全部源码）
### AsyncTask创建

```
    public AsyncTask() {
        this((Looper) null);
    }
```

```
   public AsyncTask(@Nullable Looper callbackLooper) {
   		//创建Handler，默认使用主线程的 Looper
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

		//后面这段代码看起来有点长，其实就是使用了 Future 模式，
		//先建立一个类继承 Callable 接口，再将该类赋值到 FutureTask 中，
		//至于 call() 和 done() 方法里面具体内容可以先不用理会
		//等 mFuture 被线程调用的时候，就会调用 call()
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
       			···
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
			···
            }
        };
    }
```

### InternalHandler解析

着重看下`getMainHandler()`

```
    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }
    
   private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

在创建 InternalHandler 的时候，传入了`Looper.getMainLooper()`，说明了 InternalHandler 的`handleMessage`方法可以执行 UI 操作。

我们在仔细看看`handleMessage`里面有两种处理：

```
result.mTask.finish(result.mData[0]);
```

与

```
result.mTask.onProgressUpdate(result.mData);
```

其中，result 即为 `AsyncTaskResult<?>`

```
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```

所以`result.mTask`其实就是 AsyncTask，`result.mTask.finish`就是调用 AsyncTask 的 finish 方法：

```
    private void finish(Result result) {
    //判断当前 AsyncTask 是否已被取消，已取消则调用 onCancelled，未取消则调用 onPostExecute
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

由此，可以证明`onPostExecute`在 UI 线程执行

`result.mTask.onProgressUpdate(result.mData);`就更容易理解了，直接调用`onProgressUpdate`

```
    @MainThread
    protected void onProgressUpdate(Progress... values) {
    }
```

由此，可以证明`onProgressUpdate`也在 UI 线程执行



### AsyncTask执行
asyncTask.execute(true);
     
```
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

// sDefaultExecutor 为线程池
 private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
  public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

```
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
         //判断当前 AsyncTask 的运行状态，假如运行状态为 RUNNING 或者 FINISHED，则直接报错
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

从以上代码可以得出：

- AsyncTask 内部使用了线程池进行线程操作
- **每个 AsyncTask 对象只能执行一次！！！**

？？？

使用线程池，却一个对象只能执行一次？这两个不是互相矛盾的吗？众所周知，线程池都是为了管理多线程而存在的。

我们再来仔细看下

```
 private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
  public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

`static volatile`、`static final`，说明默认实现 AsyncTask 的对象都是共用该线程池，也就是说，所有使用 AsyncTask 默认生成方式，以及继承 AsyncTask 的类使用 AsyncTask 默认生成方式，他们的线程执行都是共用一个线程池，这就为什么 AsyncTask 里面使用线程池的原因。

好了，我们重新回来看看`executeOnExecutor(Executor exec,
            Params... params)`

```
      onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);
```

首先，先执行`onPreExecute()`，其次使用线程池调用`mFuture`

关于`mFuture`，重点为`call()`

```
  mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //调用doInBackground
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                //调用postResult
                    postResult(result);
                }
                return result;
            }
        };
```

从这里可以看出，`doInBackground`是被线程池调用的时候执行的，也就是说，`doInBackground`在子线程中执行。而外，我们看看`postResult(result)`

```
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

发送了一个`MESSAGE_POST_RESULT`消息，也就是执行刚刚我们分析过的

```
        case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
```

至此，`onPreExecute`、`doInBackground`、`onPostExecute`的一个流程我们已经分析完了，大概流程如下：

execute（AsyncTask被调用的线程）-->onPreExecute（AsyncTask被调用的线程）-->doInBackground（子线程）-->onPostExecute（UI线程）

### onProgressUpdate

剩下，我们来看遗漏的`onProgressUpdate`。

首先，`onProgressUpdate`要被调用的话，需要先调用`publishProgress`：

```
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```

其实就是使用`InternalHandler`发送`MESSAGE_POST_PROGRESS`

```
             case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
```




