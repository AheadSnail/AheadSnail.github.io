---
layout: pager
title: BroadCast registerReceiver的过程分析
date: 2018-03-03 09:57:21
tags: [Android,BroadCast,registerReceiver]
description:  Android BroadCast registerReceiver的过程分析
---

Android BroadCast registerReceiver的过程分析
<!--more-->


Android BroadCast简介
```
在Android系统中，广播（Broadcast）是在组件之间传播数据（Intent）的一种机制；这些组件甚至是可以位于不同的进程中，这样它就像Binder机制一样，起到进程间通信的作用；
在Android系统中，为什么需要广播机制呢？广播机制，本质上它就是一种组件间的通信方式，如果是两个组件位于不同的进程当中，那么可以用Binder机制来实现，如果两个组件是在同一个进程中
那么它们之间可以用来通信的方式就更多了，这样看来，广播机制似乎是多余的。然而，广播机制却是不可替代的，它和Binder机制不一样的地方在于，广播的发送者和接收者事先是不需要知道对方的存在的，
这样带来的好处便是，系统的各个组件可以松耦合地组织在一起，这样系统就具有高度的可扩展性，容易与其它系统进行集成。

在软件工程中，是非常强调模块之间的高内聚低耦合性的，不然的话，随着系统越来越庞大，就会面临着越来越难维护的风险，最后导致整个项目的失败。Android应用程序的组织方式，
可以说是把这种高内聚低耦合性的思想贯彻得非常透彻，在任何一个Activity中，都可以使用一个简单的Intent，通过startActivity或者startService，
就可以把另外一个Activity或者Service启动起来为它服务，而且它根本上不依赖这个Activity或者Service的实现，只需要知道它的字符串形式的名字即可，而广播机制更绝，它连接收者的名字都不需要知道。

不过话又说回来，广播机制在Android系统中，也不算是什么创新的东西。如果读者了解J2EE或者COM，就会知道，在J2EE中，提供了消息驱动Bean（Message-Driven Bean），
用来实现应用程序各个组件之间的消息传递；而在COM中，提供了连接点（Connection Point）的概念，也是用来在应用程序各个组间间进行消息传递。无论是J2EE中的消息驱动Bean，还是COM中的连接点
，或者Android系统的广播机制，它们的实现机理都是消息发布/订阅模式的事件驱动模型，消息的生产者发布事件，而使用者订阅感兴趣的事件。

我们通过具体的例子来介绍Android系统的广播机制。在这个例子中，有一个Service，它在另外一个线程中实现了一个计数器服务，
每隔一秒钟就自动加1，然后将结果不断地反馈给应用程序中的界面线程，而界面线程中的Activity在得到这个反馈后，就会把结果显示在界面上。为什么要把计数器服务放在另外一个线程中进行呢？
我们可以把这个计数器服务想象成是一个耗时的计算型逻辑，如果放在界面线程中去实现，那么势必就会导致应用程序不能响应界面事件，最后导致应用程序产生ANR（Application Not Responding）问题。
计数器线程为了把加1后的数字源源不断地反馈给界面线程，这时候就可以考虑使用广播机制了。
```


代码案列
```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener
{

    private final static String LOG_TAG = "MainActivity";

    private Button mBtnStart;
    private Button mBtnStop;
    private TextView mTvResult;

    private ICounterService counterService = null;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        mBtnStart = findViewById(R.id.butStart);
        mBtnStop = findViewById(R.id.butStop);
        mTvResult = findViewById(R.id.tv_result);


        mBtnStop.setOnClickListener(this);
        mBtnStart.setOnClickListener(this);

        Intent bindIntent = new Intent(MainActivity.this, CounterServer.class);
        bindService(bindIntent, serviceConnection, Context.BIND_AUTO_CREATE);
        Log.i(LOG_TAG, "Main Activity Created.");
    }


    @Override
    protected void onResume()
    {
        super.onResume();
        IntentFilter counterActionFilter = new IntentFilter(CounterServer.BROADCAST_COUNTER_ACTION);
        registerReceiver(counterActionReceiver, counterActionFilter);
    }

    @Override
    protected void onDestroy()
    {
        super.onDestroy();
        unregisterReceiver(counterActionReceiver);
    }

    @Override
    public void onClick(View view)
    {
        switch (view.getId())
        {
            case R.id.butStart:
                if(counterService != null) {
                    counterService.startCounter(0);
                    mBtnStart.setEnabled(false);
                    mBtnStop.setEnabled(true);
                }
                break;

            case R.id.butStop:
                if(counterService != null) {
                    counterService.stopCounter();
                    mBtnStart.setEnabled(true);
                    mBtnStop.setEnabled(false);
                }
                break;
        }
    }

    private BroadcastReceiver counterActionReceiver = new BroadcastReceiver(){
        public void onReceive(Context context, Intent intent) {
            int counter = intent.getIntExtra(CounterServer.COUNTER_VALUE, 0);
            String text = String.valueOf(counter);
            mTvResult.setText(text);
            Log.i(LOG_TAG, "Receive counter event");
        }
    };


    private ServiceConnection serviceConnection = new ServiceConnection()
    {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder)
        {
            //获取到service的接口
            counterService = ((CounterServer.MyCounterBinder)iBinder).getCounterService();
            Log.i(LOG_TAG, "Counter Service Connected");
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName)
        {
            counterService = null;
            Log.i(LOG_TAG, "Counter Service Disconnected");
        }
    };

}

public class CounterServer extends Service implements ICounterService
{
    private final static String LOG_TAG = "zhy.broadcast.CounterService";
    public final static String BROADCAST_COUNTER_ACTION = "zhy.broadcast.COUNTER_ACTION";
    public final static String COUNTER_VALUE = "zhy.broadcast.counter.value";

    private boolean stop = false;

    private final IBinder binder = new MyCounterBinder();
    private MyAsyncTask mTask;

    @Nullable
    @Override
    public IBinder onBind(Intent intent)
    {
        return binder;
    }

    //binder对象，返回接口类型
    public class MyCounterBinder extends Binder
    {

        public ICounterService getCounterService()
        {
            return CounterServer.this;
        }
    }

    @Override
    public void onCreate()
    {
        super.onCreate();
        if(mTask == null)
        {
            mTask = new MyAsyncTask();
        }
    }

    @Override
    public void startCounter(int initVal)
    {
        mTask.execute(0);
    }


    @Override
    public void stopCounter()
    {
        stop = true;
    }

    private class MyAsyncTask extends AsyncTask<Integer, Integer, Integer>
    {
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

            int counter = values[0];
            Intent intent = new Intent(BROADCAST_COUNTER_ACTION);
            intent.putExtra(COUNTER_VALUE, counter);
            sendBroadcast(intent);
        }

        @Override
        protected void onPostExecute(Integer val) {
            int counter = val;
            Intent intent = new Intent(BROADCAST_COUNTER_ACTION);
            intent.putExtra(COUNTER_VALUE, counter);
            sendBroadcast(intent);
        }

    }
}

public interface ICounterService
{
    //开始计数
    void startCounter(int initVal);

    //停止计数
    void stopCounter();
}
```
运行结果为
![结果显示](/uploads/BrocadCast源码分析/计数.gif)

ReisterReceiver源码分析
```java
我们在代码中这样的调用 registerReceiver(counterActionReceiver, counterActionFilter);
实际调用的地方就是在ContextImpl中
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}

@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,String broadcastPermission, Handler scheduler) {
    return registerReceiverInternal(receiver, getUserId(),filter, broadcastPermission, scheduler, getOuterContext());
}
通过两个函数的中转，最终就进入到ContextImpl.registerReceiverInternal这个函数来了。这里的成员变量mPackageInfo是一个LoadedApk实例，它是用来负责处理广播的接收的，
在后面一篇文章讲到广播的发送时（sendBroadcast），会详细描述。参数broadcastPermission和scheduler都为null，
而参数context是上面的函数通过调用函数getOuterContext得到的，这里它就是指向MainActivity了，因为MainActivity是继承于Context类的，因此，这里用Context类型来引用。

private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
    }
}

由于条件mPackageInfo != null和context != null都成立，而且条件scheduler == null也成立，于是就调用mMainThread.getHandler来获得一个Handler了，这个mMainThread为ActivityThread实例对象
也即是在每一个进程中都会存在的ActivityThread对象，这个Hanlder是后面用来分发ActivityManagerService发送过的广播用的
我们先来看看ActivityThread.getHandler函数的实现，然后再回过头来继续分析ContextImpl.registerReceiverInternal函数。
public final class ActivityThread {  
    ......  
  
    final H mH = new H();  
  
    private final class H extends Handler {  
        ......  
  
        public void handleMessage(Message msg) {  
            ......  
  
            switch (msg.what) {  
            ......  
            }  
  
            ......  
        }  
  
        ......  
  
    }  
  
    ......  
  
    final Handler getHandler() {  
        return mH;  
    }  
  
    ......  
}  

再回到上一步的ContextImpl.registerReceiverInternal函数中，它通过mPackageInfo.getReceiverDispatcher函数获得一个IIntentReceiver接口对象rd，这是一个Binder对象，
接下来会把它传给ActivityManagerService，ActivityManagerService在收到相应的广播时，就是通过这个Binder对象来通知MainActivity来接收的。
我们也是先来看一下mPackageInfo.getReceiverDispatcher函数的实现，然后再回过头来继续分析ContextImpl.registerReceiverInternal函数。

public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
}
在LoadedApk.getReceiverDispatcher函数中，首先看一下参数r是不是已经有相应的ReceiverDispatcher存在了，如果有，就直接返回了，否则就新建一个ReceiverDispatcher，
并且以r为Key值保在一个HashMap中，而这个HashMap以Context，这里即为MainActivity为Key值保存在LoadedApk的成员变量mReceivers中，这样，只要给定一个Activity和BroadcastReceiver，
就可以查看LoadedApk里面是否已经存在相应的广播接收发布器ReceiverDispatcher了。

ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
            if (activityThread == null) {
                throw new NullPointerException("Handler must not be null");
            }

            mIIntentReceiver = new InnerReceiver(this, !registered);
            mReceiver = receiver;
            mContext = context;
            mActivityThread = activityThread;
            mInstrumentation = instrumentation;
            mRegistered = registered;
            mLocation = new IntentReceiverLeaked(null);
            mLocation.fillInStackTrace();
 }
 
在ReceiverDispatcher类的构造函数中，还会把传进来的Handle类型的参数activityThread保存下来，以便后面在分发广播的时候使用。以便接收广播。

final static class InnerReceiver extends IIntentReceiver.Stub {
    final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
    final LoadedApk.ReceiverDispatcher mStrongRef;

    InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
        mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
        mStrongRef = strong ? rd : null;
    }
	.....
	IIntentReceiver getIIntentReceiver() {  
        return mIIntentReceiver;  
    }  
	.....
}

在新建广播接收发布器ReceiverDispatcher时，会在构造函数里面创建一个InnerReceiver实例，这是一个Binder对象，实现了IIntentReceiver接口，
可以通过ReceiverDispatcher.getIIntentReceiver函数来获得，获得后就会把它传给ActivityManagerService，

然后继续执行ContextImpl 中的registerReceiverInternal函数的实现，也即到了
try {
        final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
            mMainThread.getApplicationThread(), mBasePackageName,
            rd, filter, broadcastPermission, userId);
        if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
        }
        return intent;
    } catch (RemoteException e) {
           throw e.rethrowFromSystemServer();
}

ActivityManagerNative.getDefault().registerReceiver()

public Intent registerReceiver(IApplicationThread caller, String packageName,
            IIntentReceiver receiver,
            IntentFilter filter, String perm, int userId) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(packageName);
        data.writeStrongBinder(receiver != null ? receiver.asBinder() : null);
        filter.writeToParcel(data, 0);
        data.writeString(perm);
        data.writeInt(userId);
        mRemote.transact(REGISTER_RECEIVER_TRANSACTION, data, reply, 0);
        reply.readException();
        Intent intent = null;
        int haveIntent = reply.readInt();
        if (haveIntent != 0) {
            intent = Intent.CREATOR.createFromParcel(reply);
        }
        reply.recycle();
        data.recycle();
        return intent;
}

这个函数通过Binder驱动程序就进入到ActivityManagerService中的registerReceiver函数中去了。
ActivityManagerService.registerReceiver

public Intent registerReceiver(IApplicationThread caller,  
            IIntentReceiver receiver, IntentFilter filter, String permission) {  
        synchronized(this) {  
            ProcessRecord callerApp = null;  
            if (caller != null) {  
                callerApp = getRecordForAppLocked(caller);  
                if (callerApp == null) {  
                    ......  
                }  
            }  
  
            List allSticky = null;  
  
            // Look for any matching sticky broadcasts...  
            Iterator actions = filter.actionsIterator();  
            if (actions != null) {  
                while (actions.hasNext()) {  
                    String action = (String)actions.next();  
                    allSticky = getStickiesLocked(action, filter, allSticky);  
                }  
            } else {  
                ......  
            }  
  
            // The first sticky in the list is returned directly back to  
            // the client.  
            Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;  
  
            ......  
  
            if (receiver == null) {  
                return sticky;  
            }  
  
            ReceiverList rl  
                = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());  
            if (rl == null) {  
                rl = new ReceiverList(this, callerApp,  
                    Binder.getCallingPid(),  
                    Binder.getCallingUid(), receiver);  
  
                if (rl.app != null) {  
                    rl.app.receivers.add(rl);  
                } else {  
                    ......  
                }  
                mRegisteredReceivers.put(receiver.asBinder(), rl);  
            }  
  
            BroadcastFilter bf = new BroadcastFilter(filter, rl, permission);  
            rl.add(bf);  
            ......  
            mReceiverResolver.addFilter(bf);  
  
            // Enqueue broadcasts for all existing stickies that match  
            // this filter.  
            if (allSticky != null) {  
                ......  
            }  
  
            return sticky;  
        }  
    } 
    ......  
}  

函数首先是获得调用registerReceiver函数的应用程序进程记录块：,也即是调用着进程的记录块
ProcessRecord callerApp = null;  
if (caller != null) {  
callerApp = getRecordForAppLocked(caller);  
if (callerApp == null) {  
    ......  
    }  
}  

接着执行
List allSticky = null;  
   // Look for any matching sticky broadcasts...  
   Iterator actions = filter.actionsIterator();  
   if (actions != null) {  
while (actions.hasNext()) {  
    String action = (String)actions.next();  
    allSticky = getStickiesLocked(action, filter, allSticky);  
}  
   } else {  
......  
   }  
   // The first sticky in the list is returned directly back to  
   // the client.  
   Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;  

这里传进来的filter只有一个action，就是前面描述的CounterService.BROADCAST_COUNTER_ACTION了，这里先通过getStickiesLocked函数查找一下有没有对应的sticky intent列表存在。
什么是Sticky Intent呢？我们在最后一次调用sendStickyBroadcast函数来发送某个Action类型的广播时，系统会把代表这个广播的Intent保存下来，这样，
后来调用registerReceiver来注册相同Action类型的广播接收器，就会得到这个最后发出的广播。这就是为什么叫做Sticky Intent了，这个最后发出的广播虽然被处理完了，
但是仍然被粘住在ActivityManagerService中，以便下一个注册相应Action类型的广播接收器还能继承处理。
这里，假设我们不使用sendStickyBroadcast来发送CounterService.BROADCAST_COUNTER_ACTION类型的广播，于是，这里得到的allSticky和sticky都为null了。

继续往下看，这里传进来的receiver不为null，于是，继续往下执行：

ReceiverList rl  = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder()); //根据receiver获取是否之前已经存在
if (rl == null) {  //如果不存在就构建一个ReceiverList，并把他加入到mRegisteredReceivers 中，key以receiver.asBinder()
	rl = new ReceiverList(this, callerApp,  Binder.getCallingPid(),  Binder.getCallingUid(), receiver);  
if (rl.app != null) {  
    rl.app.receivers.add(rl);  
} else {  
    ......  
}  
mRegisteredReceivers.put(receiver.asBinder(), rl);  
ReceiverList列表以receiver为Key值保存在ActivityManagerService的成员变量mRegisteredReceivers中，这些都是为了方便在收到广播时，快速找到对应的广播接收器的。
}     
   
   
这里其实就是把广播接收器receiver保存一个ReceiverList列表中，他的定义为 final HashMap<IBinder, ReceiverList> mRegisteredReceivers = new HashMap<>();
这个列表的宿主进程是rl.app，这里就是MainActivity所在的进程了，在ActivityManagerService中，
用一个进程记录块来表示这个应用程序进程，它里面有一个列表receivers，专门用来保存这个进程注册的广播接收器。接着，又把这个ReceiverList列表以receiver为Key值
保存在ActivityManagerService的成员变量mRegisteredReceivers中，这些都是为了方便在收到广播时，快速找到对应的广播接收器的。  

再往下看：
BroadcastFilter bf = new BroadcastFilter(filter, rl, permission);  
rl.add(bf);  
......  
mReceiverResolver.addFilter(bf);  

上面只是把广播接收器receiver保存起来了，但是还没有把它和filter关联起来，这里就创建一个BroadcastFilter来把广播接收器列表rl和filter关联起来，
然后保存在ActivityManagerService中的成员变量mReceiverResolver中去。
这样，广播接收器注册的过程就介绍完了，比较简单，但是工作又比较琐碎，主要就是将广播接收器receiver及其要接收的广播类型filter保存在ActivityManagerService中，
以便以后能够接收到相应的广播并进行处理，在下一篇文章，我们将详细分析这个过程，敬请关注。
```

RegisterRecevier流程大致为
![结果显示](/uploads/BrocadCast源码分析/Register总结.png)
