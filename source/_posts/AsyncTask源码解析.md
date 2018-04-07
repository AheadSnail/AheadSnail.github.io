---
layout: pager
title: AsyncTask源码解析
date: 2018-04-07 20:15:35
tags: [Android,AsyncTask,Handler]
description: Android AsyncTask 源码解析
---

Android AsyncTask 源码解析
<!--more-->

1.简介
```
在开发Android应用程序中，有时候我们又需要在应用程序中创建一些子线程来执行一些需要与应用程序界面进交互的计算型任务。典型的应用场景是当我们要从网上下载文件时，
为了不使主线程被阻塞，我们通常创建一个子线程来负责下载任务，同时，在下载的过程，将下载进度以百分比的形式在应用程序的界面上显示出来，这样就既不会阻塞主线程的运行，
又能获得良好的用户体验。但是，我们知道，Android应用程序的子线程是不可以操作主线程的UI的，那么，这个负责下载任务的子线程应该如何在应用程序界面上显示下载的进度呢？
如果我们能够在子线程中往主线程的消息队列中发送消息，那么问题就迎刃而解了，因为发往主线程消息队列的消息最终是由主线程来处理的，在处理这个消息的时候，
我们就可以在应用程序界面上显示下载进度了。


Android系统都为我们提供了完善的解决方案，可以使用AsyncTask类来实现，不过，为了更好地理AsyncTask类的实现，我们先来看看应用程序的主线程的消息循环模型是如何实现的。
```

```java
我们已经分析应用程序进程（主线程）的启动过程了，这里主要是针对它的消息循环模型作一个总结。当运行在Android应用程序框架层中的ActivityManagerService决定要为当前启动的应用程序创建
一个主线程的时候，它会在ActivityManagerService中的startProcessLocked成员函数调用Process类的静态成员函数start为当前应用程序创建一个主线程：

public final class ActivityManagerService extends ActivityManagerNative      
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {   
		
		
		
	private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
			
		...
		
		boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
			Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " + app.processName);
			checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            checkTime(startTime, "startProcess: returned from zygote!");
		....
	}
}

这里我们主要关注Process.start函数的第一个参数“android.app.ActivityThread”，它表示要在当前新建的线程中加载android.app.ActivityThread类，
并且调用这个类的静态成员函数main作为应用程序的入口点。ActivityThread类定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

public final class ActivityThread {    
    ......    
    
    public static final void main(String[] args) {    
        ......  
    
        Looper.prepareMainLooper();    
           
        ......    
    
        ActivityThread thread = new ActivityThread();    
        thread.attach(false);    
    
        ......   
        Looper.loop();    
    
        ......   
    
        thread.detach();    
        ......    
    }    
    ......    
}    

在这个main函数里面，除了创建一个ActivityThread实例外，就是在进行消息循环了。
在进行消息循环之前，首先会通过Looper类的静态成员函数prepareMainLooper为当前线程准备一个消息循环对象。Looper类定义在frameworks/base/core/java/android/os/Looper.java文件中：

ublic class Looper {  
    ......  
  
    // sThreadLocal.get() will return null unless you've called prepare().  
    private static final ThreadLocal sThreadLocal = new ThreadLocal();  
  
    ......  
  
    private static Looper mMainLooper = null;  
  
    ......  
  
	public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

  
    ......  
  
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
	
	public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
 
  
    public synchronized static final Looper getMainLooper() {  
        return mMainLooper;  
    }  
  
    ......  
  
    public static final Looper myLooper() {  
        return (Looper)sThreadLocal.get();  
    }  
  
    ......  
}  

Looper类的静态成员函数prepareMainLooper是专门应用程序的主线程调用的，应用程序的其它子线程都不应该调用这个函数来在本线程中创建消息循环对象，
而应该调用prepare函数来在本线程中创建消息循环对象，

为什么要为应用程序的主线程专门准备一个创建消息循环对象的函数呢？这是为了让其它地方能够方便地通过Looper类的getMainLooper函数来获得应用程序主线程中的消息循环对象。
获得应用程序主线程中的消息循环对象又有什么用呢？一般就是为了能够向应用程序主线程发送消息了。这样就可以做到在子线程里面也可以获取到这个主线程的Looper对象，这样就能往主线程发消息了

在prepareMainLooper函数中，首先会调用prepare函数在本线程中创建一个消息循环对象，然后将这个消息循环对象放在线程局部变量sThreadLocal中：
ThreadLocal.set(new Looper(quitAllowed));

接着再将这个消息循环对象保存在Looper类的静态成员变量mMainLooper中：
sMainLooper = myLooper();

消息循环对象创建好之后，回到ActivityThread类的main函数中，接下来，就是要进入消息循环了：
Looper.loop();   

 Looper类具体是如何通过loop函数进入消息循环以及处理消息队列中的消息，可以查看之前的Handler分析
 
```

2. AsyncTask源码解析
```java
前面说过，我们开发应用程序的时候，经常中需要创建一个子线程来在后台执行一个特定的计算任务，而在这个任务计算的过程中，需要不断地将计算进度或者计算结果展现在应用程序的界面中。
典型的例子是从网上下载文件，为了不阻塞应用程序的主线程，我们开辟一个子线程来执行下载任务，子线程在下载的同时不断地将下载进度在应用程序界面上显示出来，这样做出来程序就非常友好。
由于子线程不能直接操作应用程序的UI，因此，这时候，我们就可以通过往应用程序的主线程中发送消息来通知应用程序主线程更新界面上的下载进度。因为类似的这种情景在实际开发中经常碰到，
Android系统为开发人员提供了一个异步任务类（AsyncTask）来实现上面所说的功能，即它会在一个子线程中执行计算任务，同时通过主线程的消息循环来获得更新应用程序界面的机会。

为了更好地分析AsyncTask的实现，我们先举一个例子来说明它的用法

public class Counter extends Activity implements OnClickListener {  
    private final static String LOG_TAG = "shy.luo.counter.Counter";  
  
    private Button startButton = null;  
    private Button stopButton = null;  
    private TextView counterText = null;  
  
    private AsyncTask<Integer, Integer, Integer> task = null;  
    private boolean stop = false;  
  
    @Override  
    public void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.main);  
  
        startButton = (Button)findViewById(R.id.button_start);  
        stopButton = (Button)findViewById(R.id.button_stop);  
        counterText = (TextView)findViewById(R.id.textview_counter);  
  
        startButton.setOnClickListener(this);  
        stopButton.setOnClickListener(this);  
  
        startButton.setEnabled(true);  
        stopButton.setEnabled(false);  
  
  
        Log.i(LOG_TAG, "Main Activity Created.");  
    }  
  
  
    @Override  
    public void onClick(View v) {  
        if(v.equals(startButton)) {  
            if(task == null) {  
                task = new CounterTask();  
                task.execute(0);  
  
                startButton.setEnabled(false);  
                stopButton.setEnabled(true);  
            }  
        } else if(v.equals(stopButton)) {  
            if(task != null) {  
                stop = true;  
                task = null;  
  
                startButton.setEnabled(true);  
                stopButton.setEnabled(false);  
            }  
        }  
    }  
  
    class CounterTask extends AsyncTask<Integer, Integer, Integer> {  
        @Override  
        protected Integer doInBackground(Integer... vals) {  
            Integer initCounter = vals[0];  
  
            stop = false;  
            while(!stop) {  
                publishProgress(initCounter);  
  
                try {  
                    Thread.sleep(1000);  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
  
                initCounter++;  
            }  
  
            return initCounter;  
        }  
  
        @Override  
        protected void onProgressUpdate(Integer... values) {  
            super.onProgressUpdate(values);  
  
            String text = values[0].toString();  
            counterText.setText(text);  
        }  
  
        @Override  
        protected void onPostExecute(Integer val) {  
            String text = val.toString();  
            counterText.setText(text);  
        }  
    };  
}  

这个计数器程序很简单，它在界面上有两个按钮Start和Stop。点击Start按钮时，便会创建一个CounterTask实例task，然后调用它的execute函数就可以在应用程序中启动一个子线程，
并且通过调用这个CounterTask类的doInBackground函数来执行计数任务。在计数的过程中，会通过调用publishProgress函数来将中间结果传递到onProgressUpdate函数中去，
在onProgressUpdate函数中，就可以把中间结果显示在应用程序界面了。点击Stop按钮时，便会通过设置变量stop为true，这样，CounterTask类的doInBackground函数便会退出循环，
然后将结果返回到onPostExecute函数中去，在onPostExecute函数，会把最终计数结果显示在用程序界面中。

在这个例子中，我们需要注意的是：

    A. CounterTask类继承于AsyncTask类，因此它也是一个异步任务类；

    B. CounterTask类的doInBackground函数是在后台的子线程中运行的，这时候它不可以操作应用程序的界面；

    C. CounterTask类的onProgressUpdate和onPostExecute两个函数是应用程序的主线程中执行，它们可以操作应用程序的界面。


	使用AsyncTask的例子就介绍完了，下面，我们就要根据上面对AsyncTask的使用情况来重点分析它的实现了。
	
	
public abstract class AsyncTask<Params, Progress, Result> {
	
	....
	private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
	
	public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
	...
	private final AtomicBoolean mCancelled = new AtomicBoolean();
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
	
	private static InternalHandler sHandler;

    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
	
	 public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }

    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    
	}
	
	 @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
	
	@MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
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
	....
	
	 @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
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

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }

    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
	...
}

从AsyncTask的实现可以看出，当我们第一次创建一个AsyncTask对象时，首先会执行下面静态初始化代码创建一个线程池sExecutor：
	private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
	
这里的ThreadPoolExecutor是Java提供的多线程机制之一，这里用的构造函数原型为：
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,   
    BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) 
各个参数的意义如下：
    corePoolSize -- 线程池的核心线程数量

    maximumPoolSize -- 线程池的最大线程数量

    keepAliveTime -- 若线程池的线程数数量大于核心线程数量，那么空闲时间超过keepAliveTime的线程将被回收

    unit -- 参数keepAliveTime使用的时间单位

    workerQueue -- 工作任务队列

    threadFactory -- 用来创建线程池中的线程
	
简单来说，ThreadPoolExecutor的运行机制是这样的：每一个工作任务用一个Runnable对象来表示，当我们要把一个工作任务交给这个线程池来执行的时候，
就通过调用ThreadPoolExecutor的execute函数来把这个工作任务加入到线程池中去。此时，如果线程池中的线程数量小于corePoolSize，
那么就会调用threadFactory接口来创建一个新的线程并且加入到线程池中去，再执行这个工作任务；如果线程池中的线程数量等于corePoolSize，但是工作任务队列workerQueue未满，
则把这个工作任务加入到工作任务队列中去等待执行；如果线程池中的线程数量大于corePoolSize，但是小于maximumPoolSize，并且工作任务队列workerQueue已经满了，
那么就会调用threadFactory接口来创建一个新的线程并且加入到线程池中去，再执行这个工作任务；如果线程池中的线程量已经等于maximumPoolSize了，并且工作任务队列workerQueue也已经满了，
这个工作任务就被拒绝执行了。


public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
}

在AsyncTask类的构造函数里面，主要是创建了两个对象，分别是一个WorkerRunnable对象mWorker和一个FutureTask对象mFuture。
 WorkerRunnable类实现了Callable接口，此外，它的内部成员变量mParams用于保存从AsyncTask对象的execute函数传进来的参数列表：

private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
    Params[] mParams;  
}  

FutureTask类也实现了Runnable接口，所以它可以作为一个工作任务通过调用AsyncTask类的execute函数添加到sExecuto线程池中去：
public final AsyncTask<Params, Progress, Result> execute(Params... params) {  
    ......  
  
    mWorker.mParams = params;  
    sExecutor.execute(mFuture);  
  
    return this;  
}  
这里的FutureTask对象mFuture是用来封装前面的WorkerRunnable对象mWorker。当mFuture加入到线程池中执行时，它调用的是mWorker对象的call函数：
mWorker = new WorkerRunnable<Params, Result>() {  
    public Result call() throws Exception {  
           ......  
           return doInBackground(mParams);  
        }  
};  

在call函数里面，会调用AsyncTask类的doInBackground函数来执行真正的任务，这个函数是要由AsyncTask的子类来实现的，注意，这个函数是在应用程序的子线程中执行的，它不可以操作应用程序的界面。
我们可以通过mFuture对象来操作当前执行的任务，例如查询当前任务的状态，它是正在执行中，还是完成了，还是被取消了，如果是完成了，还可以通过它获得任务的执行结果，
如果还没有完成，可以取消任务的执行。

当工作任务mWorker执行完成的时候，mFuture对象中的done函数就会被被调用，根据任务的完成状况，执行相应的操作，例如，如果是因为异常而完成时，就会抛异常，如果是正常完成，
就会把任务执行结果封装成一个AsyncTaskResult对象：

private static class AsyncTaskResult<Data> {  
    final AsyncTask mTask;  
    final Data[] mData;  
  
    AsyncTaskResult(AsyncTask task, Data... data) {  
        mTask = task;  
        mData = data;  
    }  
}  

其中，成员变量mData保存的是任务执行结果，而成员变量mTask指向前面我们创建的AsyncTask对象。
最后把这个AsyncTaskResult对象封装成一个消息，并且通过消息处理器sHandler 发送消息到主线程里面，下面查看这个Hanlderd是怎么样创建的

private static Handler getHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
           sHandler = new InternalHandler();
        }
        return sHandler;
    }
}

第一次使用的时候，肯定sHandler是为空的，然后就会构建这个对象，这里查看它的构造函数

private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
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

 private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }


发现在构造函数里面，获取到的是主线程的Looper对象，这里就是在上面分析的，为什么Looper对象里面为什么要有这个mainLooper对象的存在，就是为了方便的获取到这个主线程的looper对象
所以这个sHandler发送的消息就会插到主线程里面，这里有一个好处就是，不用管你是哪里调用第一次创建这个sHandler成员变量，不管你是在哪里创建这个对象，对应的都是主线程的mLooper对象

getHandler调用的地方

   private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

	@WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

	所以为什么onProgressUpdate().onPostExecute()函数可以用来更新UI线程，这就是原因,


AsyncTask里面还有一个这样的线程池，这个线程池主要用来维护一个队列，当有任务添加进来的时候，添加到这个队列里面，当前的任务执行完毕之后，就自动的执行下一个任务

private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() { 
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();//当前的任务执行完毕之后，就安排下一个认为的执行
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
	
	
这样，AsyncTask类的主要实现就介绍完了，结合前面开发的应用程序Counter来分析，会更好地理解它的实现原理。
至此，Android应用程序线程消息循环模型就分析完成了，理解它有利于我们在开发Android应用程序时，能够充分利用多线程的并发性来提高应用程序的性能以及获得良好的用户体验。
```