---
layout: pager
title: sendBroadcast 过程分析
date: 2018-03-03 14:09:38
tags: [Android,BroadCast,sendBroadcast]
description:  Android 发送广播(sendBroadcast)过程分析
---

Android 发送广播(sendBroadcast)过程分析
<!--more-->

```
前面我们分析了Android应用程序注册广播接收器的过程，这个过程只完成了万里长征的第一步，接下来它还要等待ActivityManagerService将广播分发过来。
ActivityManagerService是如何得到广播并把它分发出去的呢？这就是本文要介绍的广播发送过程了。

广播的发送过程比广播接收器的注册过程要复杂得多了，不过这个过程仍然是以ActivityManagerService为中心。广播的发送者将广播发送到ActivityManagerService，
ActivityManagerService接收到这个广播以后，就会在自己的注册中心查看有哪些广播接收器订阅了该广播，然后把这个广播逐一发送到这些广播接收器中，
但是ActivityManagerService并不等待广播接收器处理这些广播就返回了，因此，广播的发送和处理是异步的。概括来说，广播的发送路径就是从发送者到ActivityManagerService，
再从ActivityManagerService到接收者，这中间的两个过程都是通过Binder进程间通信机制来完成的


回顾一下Android系统中的广播（Broadcast）机制简要介绍和学习计划一文中所开发的应用程序的组织架构，MainActivity向ActivityManagerService注册了一个CounterService.BROADCAST_COUNTER_ACTION
类型的计数器服务广播接收器，计数器服务CounterService在后台线程中启动了一个异步任务（AsyncTask），这个异步任务负责不断地增加计数，并且不断地将当前计数值通过广播的形式发送出去，
以便MainActivity可以将当前计数值在应用程序的界面线程中显示出来。
```

计数器服务CounterService发送广播的代码如下所示：
```java
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

在onProgressUpdate函数中，创建了一个BROADCAST_COUNTER_ACTION类型的Intent，并且在这里个Intent中附加上当前的计数器值，然后通过CounterService类的成员函数sendBroadcast将这个Intent发送出去。CounterService类继承了Service类，Service类又继承了ContextWrapper类，成员函数sendBroadcast就是从ContextWrapper类继承下来的，因此，我们就从ContextWrapper类的sendBroadcast函数开始，
分析广播发送的过程。
```

源码分析

```java
下面是在ContextWrapper中,其中mBase为Context对象，这里的实现为ContextImpl,所以进入到ContextImpl中
Context mBase;  
@Override
public void sendBroadcast(Intent intent) {
    mBase.sendBroadcast(intent);
}

@Override
public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManagerNative.getDefault().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
}

这里的resolvedType表示这个Intent的MIME类型，我们没有设置这个Intent的MIME类型，因此，这里的resolvedType为null。
接下来就调用ActivityManagerService的远程接口ActivityManagerProxy把这个广播发送给ActivityManagerService了。

class ActivityManagerProxy implements IActivityManager  
{  
    ......  
    public int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String[] requiredPermissions, int appOp, Bundle options, boolean serialized,
            boolean sticky, int userId) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeStringArray(requiredPermissions);
        data.writeInt(appOp);
        data.writeBundle(options);
        data.writeInt(serialized ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(userId);
        mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        reply.recycle();
        data.recycle();
        return res;
    }
    ...... 
}  

之后又会转发到真正提供服务的地方，即ActivityManagerService中对应的方法实现
 public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            intent = verifyBroadcastLocked(intent);

            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, bOptions, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
}

这里调用broadcastIntentLocked函数来进一步处理。
 private final int broadcastIntentLocked(ProcessRecord callerApp,  
            String callerPackage, Intent intent, String resolvedType,  
            IIntentReceiver resultTo, int resultCode, String resultData,  
            Bundle map, String requiredPermission,  
            boolean ordered, boolean sticky, int callingPid, int callingUid) {  
        intent = new Intent(intent);  
  
        ......  
  
        // Figure out who all will receive this broadcast.  
        List receivers = null;  
        List<BroadcastFilter> registeredReceivers = null;  
        try {  
            if (intent.getComponent() != null) {  
                ......  
            } else {  
                ......  
                registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false);  
            }  
        } catch (RemoteException ex) {  
            ......  
        }  
  
        final boolean replacePending =  
            (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;  
  
  
          int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        if (!ordered && NR > 0) {
            // If we are not serializing this broadcast, then send the
            // registered receivers separately so they don't wait for the
            // components to be launched.
            if (isCallerSystem) {
                checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                        isProtectedBroadcast, registeredReceivers);
            }
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                    appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                    resultExtras, ordered, sticky, false, userId);
            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing parallel broadcast " + r);
            final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
            if (!replaced) {
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
            registeredReceivers = null;
            NR = 0;
        }

      
        ......  
  
    }  
    ......  
}  

这个函数首先是根据intent找出相应的广播接收器：
 // Figure out who all will receive this broadcast.  
   List receivers = null;  
   List<BroadcastFilter> registeredReceivers = null;  
   try {  
if (intent.getComponent() != null) {  
        ......  
} else {  
    ......  
    registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false);  
}  
   } catch (RemoteException ex) {  
......  
}
   
回忆一下前面一篇文章RegisterBrocard 的过程分析中，我们将一个filter类型为BROADCAST_COUNTER_ACTION类型的BroadcastFilter实例保存在了ActivityManagerService的成员变量mReceiverResolver中，
这个BroadcastFilter实例包含了我们所注册的广播接收器，这里就通过mReceiverResolver.queryIntent函数将这个BroadcastFilter实例取回来。由于注册一个广播类型的接收器可能有多个，
所以这里把所有符合条件的的BroadcastFilter实例放在一个List中，然后返回来。在我们这个场景中，这个List就只有一个BroadcastFilter实例了，就是MainActivity注册的那个广播接收器。 
   
继续往下看：
final boolean replacePending =  (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;  
这里是查看一下这个intent的Intent.FLAG_RECEIVER_REPLACE_PENDING位有没有设置，如果设置了的话，ActivityManagerService就会在当前的系统中查看有没有相同的intent还未被处理，
如果有的话，就有当前这个新的intent来替换旧的intent。这里，我们没有设置intent的Intent.FLAG_RECEIVER_REPLACE_PENDING位，因此，这里的replacePending变量为false。

往下看：
  if (!ordered && NR > 0) {
    // If we are not serializing this broadcast, then send the
    // registered receivers separately so they don't wait for the
    // components to be launched.
    if (isCallerSystem) {
        checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                        isProtectedBroadcast, registeredReceivers);
    }
    final BroadcastQueue queue = broadcastQueueForIntent(intent);
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                    appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                    resultExtras, ordered, sticky, false, userId);
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing parallel broadcast " + r);
    final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
	
	//重点是这俩句
    if (!replaced) {
        queue.enqueueParallelBroadcastLocked(r);
        queue.scheduleBroadcastsLocked();
    }
    registeredReceivers = null;
    NR = 0;
} 

queue.enqueueParallelBroadcastLocked(r);就会执行到BroadcastQueue中对应的实现，源码的路径为
/Android7.1/frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java

public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
    mParallelBroadcasts.add(r);
    r.enqueueClockTime = System.currentTimeMillis();
}

其中mParallelBroadcasts 定义为 final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
这样，这里得到的replaced变量的值也为false，于是，就会把这个广播记录块r放在BroadcastQueue的成员变量mParcelBroadcasts中，等待进一步处理；
进一步处理的操作由函数scheduleBroadcastsLocked进行。

queue.scheduleBroadcastsLocked();就会执行到当前类中对应的方法
public void scheduleBroadcastsLocked() {
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

    if (mBroadcastsScheduled) {
        return;
    }
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
}
其中 mBroadcastsScheduled 的定义为
/**
 * Set when we current have a BROADCAST_INTENT_MSG in flight.
*/
boolean mBroadcastsScheduled = false;,他的作用为

这里的mBroadcastsScheduled表示BroadcastQueue当前是不是正在处理其它广播，如果是的话，这里就先不处理直接返回了，保证所有广播串行处理。
注意这里处理广播的方式，它是通过消息循环来处理，每当ActivityManagerService接收到一个广播时，它就把这个广播放进自己的消息队列去就完事了，根本不管这个广播后续是处理的，
因此，这里我们可以看出广播的发送和处理是异步的。（重点，这里是广播串行，异步的体现）

这里的成员变量mHandler为BroadcastQueue中的一个成员变量,是一个Handler的实例对象,
private final class BroadcastHandler extends Handler {
    public BroadcastHandler(Looper looper) {
            super(looper, null, true);
    }
	....
}

我们查看这个Handler是怎么样构建的,我们发现构造对象的方法为这个类的构造函数中,发现这里的handler.getLooper()，是非常重要的，之前有分析过Handler源码分析
BroadcastQueue(ActivityManagerService service, Handler handler,
            String name, long timeoutPeriod, boolean allowDelayBehindServices) {
        mService = service;
        mHandler = new BroadcastHandler(handler.getLooper());
        mQueueName = name;
        mTimeoutPeriod = timeoutPeriod;
        mDelayBehindServices = allowDelayBehindServices;
}

在ActivityManagerService中的 broadcastIntentLocked（）函数中有这样的执行
final BroadcastQueue queue = broadcastQueueForIntent(intent);
....      
queue.enqueueParallelBroadcastLocked(r);
queue.scheduleBroadcastsLocked();

broadcastQueueForIntent函数的实现为
BroadcastQueue broadcastQueueForIntent(Intent intent) {
    final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
    if (DEBUG_BROADCAST_BACKGROUND) Slog.i(TAG_BROADCAST,
                "Broadcast intent " + intent + " on "
                + (isFg ? "foreground" : "background") + " queue");
    return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
}
可以发现其实他就是返回一个mFgBroadcastQueue对象，或者是mBgBroadcastQueue对象，代表前台或者后台,这里不关心哪种对象,因为这俩个对象在ActivityManagerService中的构造为下面的语句
mFgBroadcastQueue = new BroadcastQueue(this, mHandler, "foreground", BROADCAST_FG_TIMEOUT, false);
mBgBroadcastQueue = new BroadcastQueue(this, mHandler,"background", BROADCAST_BG_TIMEOUT, true);

这里的mHandler为 ActivityMangerServife 中的  final MainHandler mHandler;也即是在主线程的Handler，
mHandlerThread = new ServiceThread(TAG,android.os.Process.THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
mHandlerThread.start();
//这里获取到Looper对象
mHandler = new MainHandler(mHandlerThread.getLooper());

new ServiceThread,本质是继承于HandelrThread，至于为什么要使用HandlerThread在之前的Handler源码分析中有体现过，也即是为了下面获取到Looper的时候不为空
public class ServiceThread extends HandlerThread {
    private static final String TAG = "ServiceThread";

    private final boolean mAllowIo;

    public ServiceThread(String name, int priority, boolean allowIo) {
        super(name, priority);
        mAllowIo = allowIo;
 } 
 ....
 所以这里就关系了ActivityManagerService的构建的时机，在之前的源码分析中 也即是在SystemService，具体可以去查看，run方法中有这样的实现Looper.prepareMainLooper();,所以是为主线程的Looper对象
 try {
    ......
    // Mmmmmm... more memory!
    VMRuntime.getRuntime().clearGrowthLimit();
	//设置线程的优先级
    // Prepare the main looper thread (this thread).
    android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
	android.os.Process.setCanSelfBackground(false);
    
	准备Main loop对象
	Looper.prepareMainLooper();
    // Initialize native services.
	初始话 native services
    System.loadLibrary("android_servers");
    // Check whether we failed to shut down last time we tried.
    // This call may not return.
    performPendingShutdown();
    // Initialize the system context.
	初始话系统的 上下文对象
    createSystemContext();
    // Create the system service manager.
    创建系统的 service manager,用来启动系统的服务
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
 } finally {
    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
}
}

BroadcastQueue(ActivityManagerService service, Handler handler,
            String name, long timeoutPeriod, boolean allowDelayBehindServices) {
        mService = service;
        mHandler = new BroadcastHandler(handler.getLooper());所以这边获取到的是主线程的Looper对象
        mQueueName = name;
        mTimeoutPeriod = timeoutPeriod;
        mDelayBehindServices = allowDelayBehindServices;
}


继续分析
mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
通过它的sendMessage函数把一个类型为BROADCAST_INTENT_MSG的消息，我们查看消息的处理
case BROADCAST_INTENT_MSG: {
    if (DEBUG_BROADCAST) Slog.v(AG_BROADCAST, "Received BROADCAST_INTENT_MSG");
    processNextBroadcast(true);
} break;

private final void processNextBroadcast(boolean fromMsg) {  
    synchronized(this) {  
            BroadcastRecord r;  
  
            ......  
  
            if (fromMsg) {  
                mBroadcastsScheduled = false;  
            }  
  
            // First, deliver any non-serialized broadcasts right away.  
            while (mParallelBroadcasts.size() > 0) {  
                r = mParallelBroadcasts.remove(0);  
                ......  
                final int N = r.receivers.size();  
                ......  
                for (int i=0; i<N; i++) {  
                    Object target = r.receivers.get(i);  
                    ......  
  
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);  
                }  
                addBroadcastToHistoryLocked(r);  
                ......  
            }  

         ......  
 }  
这里传进来的参数fromMsg为true，于是把mBroadcastScheduled重新设为false，这样，下一个广播就能进入到消息队列中进行处理了。前面中，
把一个广播记录块BroadcastRecord放在了mParallelBroadcasts中，因此，这里就把它取出来进行处理了。广播记录块BroadcastRecord的receivers列表中包含了要接收这个广播的目标列表
即前面我们注册的广播接收器，用BroadcastFilter来表示，这里while循环中的for循环就是把这个广播发送给每一个订阅了该广播的接收器了，通过deliverToRegisteredReceiverLocked函数执行。

private final void deliverToRegisteredReceiverLocked(BroadcastRecord r,  
            BroadcastFilter filter, boolean ordered) {  
    boolean skip = false;  
    if (filter.requiredPermission != null) {  
            ......  
        }  
        if (r.requiredPermission != null) {  
            ......  
        }  
  
        if (!skip) {  
            // If this is not being sent as an ordered broadcast, then we  
            // don't want to touch the fields that keep track of the current  
            // state of ordered broadcasts.  
            if (ordered) {  
                ......  
            }  
  
            try {  
                ......  
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,  
                    new Intent(r.intent), r.resultCode,  
                    r.resultData, r.resultExtras, r.ordered, r.initialSticky);  
                ......  
            } catch (RemoteException e) {  
                ......  
            }  
        }  
}  

函数首先是检查一下广播发送和接收的权限，在我们分析的这个场景中，没有设置权限，因此，这个权限检查就跳过了，这里得到的skip为false，于是进入下面的if语句中。
由于上面传时来的ordered参数为false，因此，直接就调用performReceiveLocked函数来进一步执行广播发送的操作了。

void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        if (app != null) {
            if (app.thread != null) {
                // If we have an app thread, do the call through that so it is
                // correctly ordered with other one-way calls.
                try {
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                            data, extras, ordered, sticky, sendingUser, app.repProcState);
                // TODO: Uncomment this when (b/28322359) is fixed and we aren't getting
                // DeadObjectException when the process isn't actually dead.
                //} catch (DeadObjectException ex) {
                // Failed to call into the process.  It's dying so just let it die and move on.
                //    throw ex;
                } catch (RemoteException ex) {
                    // Failed to call into the process. It's either dying or wedged. Kill it gently.
                    synchronized (mService) {
                        Slog.w(TAG, "Can't deliver broadcast to " + app.processName
                                + " (pid " + app.pid + "). Crashing it.");
                        app.scheduleCrash("can't deliver broadcast");
                    }
                    throw ex;
                }
            } else {
                // Application has died. Receiver doesn't exist.
                throw new RemoteException("app.thread must not be null");
            }
        } else {
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
	
注意，这里传进来的参数app是注册广播接收器的Activity所在的进程记录块，在我们分析的这个场景中，由于是MainActivity调用registerReceiver函数来注册这个广播接收器的，
因此，参数app所代表的ProcessRecord就是MainActivity所在的进程记录块了；而参数receiver也是注册广播接收器时传给ActivityManagerService的一个Binder对象，它的类型是IIntentReceiver，
具体可以参考上一篇文章Android应用程序注册广播接收器（registerReceiver）的过程


执行 app.thread.scheduleRegisteredReceiver ApplicationThreadProxy的源码路径为/Android7.1/frameworks/base/core/java/android/app/ApplicationThreadNative.java
class ApplicationThreadProxy implements IApplicationThread {  
	....
	public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
            int resultCode, String dataStr, Bundle extras, boolean ordered,
            boolean sticky, int sendingUser, int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(receiver.asBinder());
        intent.writeToParcel(data, 0);
        data.writeInt(resultCode);
        data.writeString(dataStr);
        data.writeBundle(extras);
        data.writeInt(ordered ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(sendingUser);
        data.writeInt(processState);
        mRemote.transact(SCHEDULE_REGISTERED_RECEIVER_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
	...
	}
	
然后就跳到了最终的实现类里面即在ActivityThread中
// This function exists to make sure all receiver dispatching is
        // correctly ordered, since these are one-way calls and the binder driver
        // applies transaction ordering per object for such calls.
        public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
        updateProcessState(processState, false);
        receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
}

这里的receiver是在前面一篇文章Android应用程序注册广播接收器（registerReceiver）的过程分析，它的具体类型是LoadedApk.ReceiverDispatcher.InnerReceiver，
即定义在LoadedApk类的内部类ReceiverDispatcher里面的一个内部类InnerReceiver，这里调用它的performReceive函数。

static final class ReceiverDispatcher {
    final static class InnerReceiver extends IIntentReceiver.Stub {
	....
	@Override
	public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                final LoadedApk.ReceiverDispatcher rd;
                if (intent == null) {
                    Log.wtf(TAG, "Null intent received");
                    rd = null;
                } else {
                    rd = mDispatcher.get();
                }
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Receiving broadcast " + intent.getAction()
                            + " seq=" + seq + " to " + (rd != null ? rd.mReceiver : null));
                }
                if (rd != null) {
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                } else {
                    // The activity manager dispatched a broadcast to a registered
                    // receiver in this process, but before it could be delivered the
                    // receiver was unregistered.  Acknowledge the broadcast on its
                    // behalf so that the system's broadcast sequence can continue.
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to unregistered receiver");
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    try {
                        if (extras != null) {
                            extras.setAllowFds(false);
                        }
                        mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
                    } catch (RemoteException e) {
                        throw e.rethrowFromSystemServer();
			}
		}
	}
	.....
}

这里，它只是简单地调用ReceiverDispatcher的performReceive函数来进一步处理，这里的ReceiverDispatcher类是LoadedApk类里面的一个内部类
 public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            if (intent == null) {
                Log.wtf(TAG, "Null intent received");
            } else {
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                }
            }
            if (intent == null || !mActivityThread.post(args)) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    args.sendFinished(mgr);
                }
            }
        }
		
这里mActivityThread成员变量的类型为Handler，它是前面MainActivity注册广播接收器时，从ActivityThread取得的，具体可以参考前面一篇文章Android应用程序注册广播接收器（registerReceiver）
这里ReceiverDispatcher借助这个Handler，把这个广播以消息的形式放到MainActivity所在的这个ActivityThread的消息队列中去，因此，ReceiverDispatcher不等这个广播被MainActivity处理就返回了
这里也体现了广播的发送和处理是异步进行的。

注意这里处理消息的方式是通过Handler.post函数进行的，post函数的参数是Runnable类型的，这个消息最终会调用这个这个参数的run成员函数来处理。
这里的Args类是LoadedApk类的内部类ReceiverDispatcher的一个内部类，它继承于Runnable类，因此，可以作为mActivityThread.post的参数传进去，代表这个广播的intent也保存在这个Args实例中。

这个函数定义在frameworks/base/core/java/android/os/Handler.java文件中，这个函数我们就不看了，有兴趣的读者可以自己研究一下，它的作用就是把消息放在消息队列中，
然后就返回了，这个消息最终会在传进来的Runnable类型的参数的run成员函数中进行处

final class Args extends BroadcastReceiver.PendingResult implements Runnable {
	.....
	  public void run() {
        final BroadcastReceiver receiver = mReceiver;
        ....
          
        try {
            ClassLoader cl =  mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            intent.prepareToEnterProcess();
            setExtrasClassLoader(cl);
            receiver.setPendingResult(this);
            receiver.onReceive(mContext, intent);
        } catch (Exception e) {
           ......
        }
	....
}

这里的mReceiver是ReceiverDispatcher类的成员变量，它的类型是BroadcastReceiver，这里它就是MainActivity注册广播接收器时创建的BroadcastReceiver实例了，
具体可以参考前面一篇文章Android应用程序注册广播接收器（registerReceiver）的过程分析

有了这个ReceiverDispatcher实例之后，就可以调用它的onReceive函数把这个广播分发给它处理了。，也即回到了我们MainActiivy中的onReceiver接受了
public class MainActivity extends Activity implements OnClickListener {      
    ......    
  
    private BroadcastReceiver counterActionReceiver = new BroadcastReceiver(){    
        public void onReceive(Context context, Intent intent) {    
            int counter = intent.getIntExtra(CounterService.COUNTER_VALUE, 0);    
            String text = String.valueOf(counter);    
            counterText.setText(text);    
  
            Log.i(LOG_TAG, "Receive counter event");    
        }      
    }  
    ......    
}  

```

最后，我们总结一下这个Android应用程序发送广播的过程：

1.计数器服务CounterService通过sendBroadcast把一个广播通过Binder进程间通信机制发送给ActivityManagerService，ActivityManagerService根据这个广播的Action类型找到相应的广播接收器，
然后把这个广播放进自己的消息队列中去，就完成第一阶段对这个广播的异步分发了；

2. ActivityManagerService在消息循环中处理这个广播，并通过Binder进程间通信机制把这个广播分发给注册的广播接收分发器ReceiverDispatcher，
ReceiverDispatcher把这个广播放进MainActivity所在的线程的消息队列中去，就完成第二阶段对这个广播的异步分发了；

3. ReceiverDispatcher的内部类Args在MainActivity所在的线程消息循环中处理这个广播，最终是将这个广播分发给所注册的BroadcastReceiver实例的onReceive函数进行处理。

sendBrocard流程大致为
![结果显示](/uploads/BrocadCast源码分析/sendBrocard流程.png)
