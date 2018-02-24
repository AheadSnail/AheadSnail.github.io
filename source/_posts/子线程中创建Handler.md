---
layout: pager
title: 子线程中创建Handler
date: 2018-02-23 23:04:45
tags: [Android,Hanler]
description:  Android 子线程里面使用Handler
---

Android 子线程里面使用Handler
<!--more-->


```
更新ui...等，那么反过来，怎么才能让主线程给子线程发消息，通知子线程做一些耗时逻辑？？之前的学习我们知道，Android的消息机制遵循三个步骤：
1　　创建当前线程的Looper　　

2　　创建当前线程的Handler　

3　　调用当前线程Looper对象的loop方法
看过之前文章的朋友会注意到，本篇我特意强调了“当前线程”。是的之前我们学习的很多都是Android未我们做好了的，譬如：创建主线程的Looper、主线程的消息队列...
就连我们使用的handler也是主线程的。那么如果我想创建非主线程的Handler并且发送消息、处理消息，
这一系列的操作我们应该怎么办那？？？不怎么办、凉拌～～～什么意思？？？依葫芦画瓢，依然遵循上面的三步走，直接上代码！！！！
```

案例使用
```java
public class MainActivity extends AppCompatActivity
{
    private MyThread mThread;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        //开启线程
        mThread = new MyThread();
        mThread.start();

        //handler就跟Thread的looper关联起来了
        Handler threadHandler = new Handler(mThread.mLooper)
        {
            @Override
            public void handleMessage(Message msg)
            {
                super.handleMessage(msg);
                Log.d("HandlerThreadActivity", "uiThread2------"+Thread.currentThread());//子线程
            }
        };

        threadHandler.sendEmptyMessage(0);
        Log.d("HandlerThreadActivity","uiThread1------"+Thread.currentThread());//主线程
    }

    private class MyThread extends Thread
    {
        public Looper mLooper;

        @Override
        public void run()
        {
            Looper.prepare();//第一步准备当前线程的loop对象，已经Loop对象里面的成员变量MessageQueue对象
            mLooper = Looper.myLooper();//获取到当前线程的loop对象，因为上一步骤已经准备好了，详解可以看上一篇
            Looper.loop();//当前Looper开始循环,原理的实现会获取到当前线程的Looper对象已经MessageQueue对象，也就是我们在子线程准备好的Looper.prepare对象
        }
    }
}

分析下handler是怎么样关联looper的
Handler threadHandler = new Handler(thread.getLooper())
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;//直接将子线程的Looper对象存储起来
    mQueue = looper.mQueue;//将子线程创建的Looper对象中的MessageQueue对象存储起来,最终通过发送消息，将msg存储在MessageQueue中，然后通过调用Looper.looper无限的循环获取消息
    mCallback = callback;
    mAsynchronous = async;
}

原先通知直接的new Handler的化，就是通过Looper.myLooper获取到当前线程的looper对象，这里不能通过这样的方式获取到，因为这样获取到的是主线程里面的Looper对象，也即是ActivityThread
中调用prepareMainLooper准备的对象，这里的MessageQueue也是主线程的，如果通过这样的方式来使用的化，通过这个handler发送的消息，就会由主线程的Looper循环获取到，就不是由子线程的Looper
对象所获取到，而通过直接传递管理子线程的Looper对象，那么自然就关联起来了
mLooper = Looper.myLooper();
if (mLooper == null) {
    throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
}
mQueue = mLooper.mQueue;

```
结果显示如下，会随机的产生空指针
![结果显示](/uploads/Handler子线程里面使用/Handler使用失败.png)

```
代码如上，我们依然循序Android的三步走战略，完成了子线程Handler的创建，难道这样创建完了，就可以发消息了么？发的消息在什么线程处理？一系列的问题，怎么办？看代码！！！运行上述代码，]
我们发现一个问题，就是此代码一会崩溃、一会不崩溃，通过查看日志我们看到崩溃的原因是空指针。谁为空？？？查到是我们的Looper对象，怎么会那？
我不是在子线程的run方法中初始化Looper对象了么？话是没错，但是你要知道，当你statr子线程的时候，虽然子线程的run方法得到执行，但是主线程中代码依然会向下执行，
造成空指针的原因是当我们new Handler(childThread.childLooper)的时候，run方法中的Looper对象还没初始化。当然这种情况是随机的，所以造成偶现的崩溃。
那怎么办？难道我们不能创建子线程Handler ？？？No!!!No!!!No!!!，你能想到的Android早就为我们实现好了，HandlerThread类就是解决这个问题的关键所在，看代码！！！
```

```java
 HandlerThread thread = new HandlerThread("HandlerThread");
 thread.start();

//handler就跟Thread的looper关联起来了
Handler threadHandler = new Handler(thread.getLooper())
{
    @Override
    public void handleMessage(Message msg)
    {
        super.handleMessage(msg);
        Log.d("HandlerThreadActivity", "uiThread2------"+Thread.currentThread());//子线程
    }
};

    threadHandler.sendEmptyMessage(0);
    Log.d("HandlerThreadActivity","uiThread1------"+Thread.currentThread());//主线程
```
结果显示如下，不管怎么样运行都是成功的
![结果显示](/uploads/Handler子线程里面使用/Handler使用成功.png)

分析下使用HanlderThread为什么可以,它到底是什么
```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

	@Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();//thread.getLooper() 如果thread.getLooper 中如果线程还没有运行，或者mLooper对象为空，那么就会wait,cpu占用资源，等待，到了这里
			//说明线程已经运行了，而且Looper也已经准备好了，那么就可以唤醒了，调用了notifyAll()函数唤醒，那么那边获取到的mLooper对象就不会为空，那么此时就不会崩溃
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    ....
}

通过上面分析可以发现，其实他就是一个Thread，而且当线程运行的时候，也是通过Looper.prepare(),然后获取到当前线程的Looper对象，然后通过Looper.loop 三个步骤来使用的，跟我们自己写的
Thread是一样的，只是在获取到thread.getLooper()对象的时候，有点不同
Handler threadHandler = new Handler(thread.getLooper())
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
        
    // If the thread has been started, wait until the looper has been created.
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
HandlerThread类的getLooper方法如上，我们看到当我们获取当前线程Looper对象的时候，会先判断当前线程是否存活，然后还要判断Looper对象是否为空，都满足之后才会返回给我Looper对象，
否则处于等待状态！！既然有等待，那就有唤醒的时候，在那里那？？？我们发现HandlerThread的run方法中，有如下代码：
```

参照HandlerThread的实现原理，修改我们自己dThread，做到同样的效果
```java
public class MainActivity extends AppCompatActivity
{
	private MyThread mThread;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //开启线程
        mThread = new MyThread();
        mThread.start();

        //HandlerThread thread = new HandlerThread("HandlerThread");
        //thread.start();

        //handler就跟Thread的looper关联起来了
        Handler threadHandler = new Handler(mThread.getLooper())
        {
            @Override
            public void handleMessage(Message msg)
            {
                super.handleMessage(msg);
                Log.d("HandlerThreadActivity", "uiThread2------"+Thread.currentThread());//子线程
            }
        };

        threadHandler.sendEmptyMessage(0);
        Log.d("HandlerThreadActivity","uiThread1------"+Thread.currentThread());//主线程
    }

    private class MyThread extends Thread
    {
        private Looper mLooper;

        //获取到Looper对象
        public Looper getLooper() {
            if (!isAlive()) {
                return null;
            }
            // If the thread has been started, wait until the looper has been created.
            //如果线程已经开始了，就等待mLooper对象的赋值，如果为空，调用wait（）函数，阻塞,直到有人唤醒
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

        @Override
        public void run()
        {
            Looper.prepare();
            synchronized (this) {
                mLooper = Looper.myLooper();
                notifyAll();//如果线程准备好了，Looper对象也已经赋值好了，就唤醒
            }
            Looper.loop();
        }
}


```
结果显示如下，不管怎么样运行都是成功的
![结果显示](/uploads/Handler子线程里面使用/Handler使用成功.png)