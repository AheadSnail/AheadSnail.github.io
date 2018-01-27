---
layout: pager
title: startService启动分析
date: 2018-01-26 15:30:26
tags: [Android,Service,startService]
description:  Android startService启动分析
---

Android startService启动分析
<!--more-->

ActivityManagerService的启动
```java
这里先介绍一下ActivityManagerService，这个ActivityManagerService类实现在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中，
它是Binder进程间通信机制中的Server角色，它是随机启动的。随机启动的Server是在frameworks/base/services/java/com/android/server/SystemServer.java文件里面进行启动的，
我们来看一下ActivityManagerService启动相关的代码：

mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
而关于SystemServiceManager是怎么样找到对应的类，并得到这个对象的，可以自行查看，源码的路径为frameworks/base/services/java/com/android/server/SystemServerManager.java
其实就是通过loadClasss()，然后newInstance得到对应的对象，并且会调用对应onStart()函数，之前已经分析过..所以会执行Lifecycle构造函数，并执行onStart函数，下面为Lifecycle声明

public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
}

通过上面构造得到ActivityManagerService对象之后，通过调用下面的方法，将当前的service，交给ServiceManager,至于为什么要交给ServcieManager管理，这里就要去了解IPC的通信机制了
// Set up the Application instance for the system process and get started.
mActivityManagerService.setSystemProcess();
public void setSystemProcess() {
try {
.....
//public static final String ACTIVITY_SERVICE = "activity";
ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
....
}
```


客户端的代码编写
```java
package shy.luo.ashmem;    
......  
  
public class Client extends Activity implements OnClickListener {  
    ......  
    IMemoryService memoryService = null;  
    ......  
  
    @Override  
    public void onCreate(Bundle savedInstanceState) {  
        ......  
  
        IMemoryService ms = getMemoryService();  
        if(ms == null) {          
            startService(new Intent("shy.luo.ashmem.server"));  
        } else {  
            Log.i(LOG_TAG, "Memory Service has started.");  
        }  
  
        ......  
  
        Log.i(LOG_TAG, "Client Activity Created.");  
    }  
  
    ......  
}  

这里的“shy.luo.ashmem.server”是在程序配置文件AndroidManifest.xml配置的Service的名字，用来告诉Android系统它所要启动的服务的名字：

<manifest xmlns:android="http://schemas.android.com/apk/res/android"  
    package="shy.luo.ashmem"  
    android:sharedUserId="android.uid.system"  
    android:versionCode="1"  
    android:versionName="1.0">  
        <application android:icon="@drawable/icon" android:label="@string/app_name">  
            ......  
            <service   
                android:enabled="true"   
                android:name=".Server"  
                android:process=".Server" >  
                    <intent-filter>  
                        <action android:name="shy.luo.ashmem.server"/>  
                        <category android:name="android.intent.category.DEFAULT"/>  
                    </intent-filter>  
            </service>  
        </application>  
</manifest>   

这里，名字“shy.luo.ashmem.server”对应的服务类为shy.luo.ashmem.Server，下面语句：
startService(new Intent("shy.luo.ashmem.server"));  
就表示要在一个新的进程中启动shy.luo.ashmem.Server这个服务类，它必须继承于Android平台提供的Service类：

Service代码编写
package shy.luo.ashmem; 
......  
  
public class Server extends Service {  
      
    ......  
  
    @Override  
    public IBinder onBind(Intent intent) {  
            return null;  
    }  
  
    @Override  
    public void onCreate() {  
        ......  
  
    }  
  
    ......  
}  
```

源码分析
```java

当我们调用 startService(new Intent("shy.luo.ashmem.server"));  就会调用到ContextImpl中的startService函数
@Override
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, mUser);
}

private ComponentName startServiceCommon(Intent service, UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        ComponentName cn = ActivityManagerNative.getDefault().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());
        if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException(
                            "Not allowed to start service " + service
                            + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException(
                            "Unable to start service " + service
                            + ": " + cn.getClassName());
                }
           }
            return cn;
		} catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
    }
}

ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());
上面设计到了进程间的通信，其实是获取到的对象是ActivityManagerProxy对象，所以就会调用对应的startService方法，至于为什么得到这个对象之前分析过。。
源码的路径为 /frameworks/base/core/java/android/app/ActivityManagerNative 的内部类里面,下面为他的函数实现


public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    service.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeString(callingPackage);
    data.writeInt(userId);
    mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
    reply.readException();
    ComponentName res = ComponentName.readFromParcel(reply);
    data.recycle();
    reply.recycle();
    return res;
}

参数service是一个Intent实例，它里面指定了要启动的服务的名称，就是前面我们所说的“shy.luo.ashmem.server”了。
参数caller是一个IApplicationThread实例，它是一个在主进程创建的一个Binder对象。在Android应用程序中，每一个进程都用一个ActivityThread实例来表示，
而在ActivityThread类中，有一个成员变量mAppThread，它是一个ApplicationThread实例，实现了IApplicationThread接口，它的作用是用来辅助ActivityThread类来执行一些操作，
这个我们在后面会看到它是如何用来启动服务的。

参数resolvedType是一个字符串，它表示service这个Intent的MIME类型，它是在解析Intent时用到的。在这个例子中，我们没有指定这个Intent 的MIME类型，因此，这个参数为null。
ActivityManagerProxy类的startService函数把这三个参数写入到data本地变量去，接着通过mRemote.transact函数进入到Binder驱动程序，
然后Binder驱动程序唤醒正在等待Client请求的ActivityManagerService进程，最后进入到ActivityManagerService的startService函数中。

这里的mRemote对象也即是远程的提供服务的ActivityManagerService，所以就会调用他的onTransact函数
mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);下面是onTransact函数中对应的START_SERVICE_TRANSACTION实现

case START_SERVICE_TRANSACTION: {
    data.enforceInterface(IActivityManager.descriptor);
    IBinder b = data.readStrongBinder();
    IApplicationThread app = ApplicationThreadNative.asInterface(b);
    Intent service = Intent.CREATOR.createFromParcel(data);
    String resolvedType = data.readString();
    String callingPackage = data.readString();
    int userId = data.readInt();
    ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);
    reply.writeNoException();
    ComponentName.writeToParcel(cn, reply);
    return true;
}

ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);
这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件							
@Override
public ComponentName startService(IApplicationThread caller, Intent service,
    String resolvedType, String callingPackage, int userId)
    throws TransactionTooLargeException {
    enforceNotIsolatedCaller("startService");
    // Refuse possible leaked file descriptors
    if (service != null && service.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    if (callingPackage == null) {
        throw new IllegalArgumentException("callingPackage cannot be null");
    }
    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
        "startService: " + service + " type=" + resolvedType);
    synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid, callingPackage, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
    }
}

这里的参数caller、service和resolvedType分别对应ActivityManagerProxy.startService传进来的三个参数。
ActivityManagerService.startServiceLocked
这个函数同样定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：		
public final class ActivityManagerService extends ActivityManagerNative  
                           implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
  
    ......  
    ComponentName startServiceLocked(IApplicationThread caller,  
            Intent service, String resolvedType,  
            int callingPid, int callingUid) {  
        synchronized(this) {  
            ......  
  
            ServiceLookupResult res =  
                retrieveServiceLocked(service, resolvedType,  
                callingPid, callingUid);  
              
            ......  
              
            ServiceRecord r = res.record;  
              
            ......  
  
            return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        }  
    }  
    ......  
}  

函数首先通过retrieveServiceLocked来解析service这个Intent，就是解析前面我们在AndroidManifest.xml定义的Service标签的intent-filter相关内容，
private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal) {
		....
		// TODO: come back and remove this assumption to triage all services
		//这下面就是解析Intent的参数，这里也利用了进程间的通信，调用了PackageService对应的方法，来解析，并将解析的结果保存在ResolveInfo中
        ResolveInfo rInfo = AppGlobals.getPackageManager().resolveService(service,
                        resolvedType, ActivityManagerService.STOCK_PM_FLAGS
                                | PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                        userId);
		....

}

然后将解析结果放在res.record中，然后继续调用startServiceInnerLocked进一步处理。
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
		....
		String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
		....

}		
	
ActivityManagerService.bringUpServiceLocked
这个函数同样定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：
public final class ActivityManagerService extends ActivityManagerNative  
                            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
  
    ......  
  
    private final boolean bringUpServiceLocked(ServiceRecord r,  
                    int intentFlags, boolean whileRestarting) {  
  
        ......  
  
        final String appName = r.processName;  
  
        ......  
  
        // Not running -- get it started, and enqueue this service record  
        // to be executed when the app comes up.  
        if (mAm.startProcessLocked(appName, r.appInfo, true, intentFlags,  
                    "service", r.name, false) == null) {  
  
            ......  
  
            return false;  
        }  
  
        if (!mPendingServices.contains(r)) {  
            mPendingServices.add(r);  
        }  
  
        return true;  
  
    }  
    ......  
}  

这里的appName便是我们前面在AndroidManifest.xml文件定义service标签时指定的android:process属性值了，即“.Server”。
接着调用startProcessLocked函数来创建一个新的进程，以便加载自定义的Service类。最后将这个ServiceRecord保存在成员变量mPendingServices列表中，后面会用到。
mAm.startProcessLocked
这个函数同样定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：	
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {

		.....
		startProcessLocked(app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
        checkTime(startTime, "startProcess: done starting proc!");
        return (app.pid != 0) ? app : null;
}

这个函数同样定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：这个函数有三个重载函数
private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
		....
        try {  
  
            ......  
  
            int pid = Process.start("android.app.ActivityThread",  
                            mSimpleProcessManagement ? app.processName : null, uid, uid,  
                            gids, debugFlags, null);  
  
            ......  
  
            if (pid == 0 || pid == MY_PID) {  
                  
                ......  
  
            } else if (pid > 0) {  
                app.pid = pid;  
                app.removed = false;  
                synchronized (mPidsSelfLocked) {  
                    this.mPidsSelfLocked.put(pid, app);  
                    ......  
                }  
            } else {  
                  
                ......  
            }  
  
        } catch (RuntimeException e) {  
  
            ......  
  
    }  
}
	
这里调用Process.start函数创建了一个新的进程，指定新的进程执行android.app.ActivityThread类。最后将表示这个新进程的ProcessRecord保存在mPidSelfLocked列表中，后面会用到。
Process.start

这个函数定义在frameworks/base/core/java/android/os/Process.java文件中，这个函数我们就不看了，有兴趣的读者可以自己研究一下。在这个场景中，它就是新建一个进程，
然后导入android.app.ActivityThread这个类，然后执行它的main函数。

ActivityThread.main
这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：

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
} 

注意，执行到这里的时候，已经是在上一步创建的新进程里面了，即这里的进程是用来启动服务的，原来的主进程已经完成了它的命令，返回了。
所以，在Android应用程序中，每一个进程对应一个ActivityThread实例，所以，这个函数会创建一个thread实例，然后调用ActivityThread.attach函数进一步处理。
ActivityThread.attach
这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中： 

public final class ActivityThread {  
      
    ......  
  
    private final void attach(boolean system) {  
          
        ......  
  
        if (!system) {  
  
            ......  
  
            IActivityManager mgr = ActivityManagerNative.getDefault();  
            try {  
                mgr.attachApplication(mAppThread);  
            } catch (RemoteException ex) {  
            }  
        } else {  
          
            ......  
  
        }  
  
        ......  
  
    }  
  
    ......  
  
} 

这里传进来的参数system为false。成员变量mAppThread是一个ApplicationThread实例，这个实例在new ActivityThread的时候就创建了
final ApplicationThread mAppThread = new ApplicationThread();我们在前面已经描述过这个实例的作用，它是用来辅助ActivityThread来执行一些操作的。
调用ActivityManagerNative.getDefault函数得到ActivityManagerService的远程接口，即ActivityManagerProxy，接着调用它的attachApplication函数。

ActivityManagerProxy.attachApplication
这个函数定义在frameworks/base/core/java/android/app/ActivityManagerNative.java文件中：
public void attachApplication(IApplicationThread app) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(app.asBinder());
    mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
} 

这个函数主要是将新进程里面的IApplicationThread实例通过Binder驱动程序传递给ActivityManagerService。
ActivityManagerService.attachApplication
这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：
 @Override
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
ActivityManagerService.attachApplicationLocked
这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：
public final class ActivityManagerService extends ActivityManagerNative  
                        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
  
    ......  
  
    private final boolean attachApplicationLocked(IApplicationThread thread,  
            int pid) {  
	
	......  
    String processName = app.processName;  
          
    ......  
  
    app.thread = thread;  
  
    ......  
          
    boolean badApp = false;  
	......
    thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
	......
	// Find any services that should be running in this process...
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
            badApp = true;
        }
    }
    ......  
} 

先执行thread.bindApplication，这里的thread为ApplicationThread，也即是在创建ActivityThread 创建的的对象，所以会执行到对应的方法
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {
	.....
	sendMessage(H.BIND_APPLICATION, data);
}

 case BIND_APPLICATION:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
    AppBindData data = (AppBindData)msg.obj;
    handleBindApplication(data);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;

private void handleBindApplication(AppBindData data) {
	....
	/ Continue loading instrumentation.
    if (ii != null) {
        final ApplicationInfo instrApp = new ApplicationInfo();
        ii.copyTo(instrApp);
        instrApp.initForUser(UserHandle.myUserId());
        final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
        final ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

        try {
			//使用类加载器，加载并创建Instrumentation对象
            final ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
        }

        final ComponentName component = new ComponentName(ii.packageName, ii.name);
		
		//初始化Instrumentation
        mInstrumentation.init(this, instrContext, appContext, component,
                    data.instrumentationWatcher, data.instrumentationUiAutomationConnection);

        if (mProfiler.profileFile != null && !ii.handleProfiling
                    && mProfiler.profileFd == null) {
                mProfiler.handlingProfiling = true;
                final File file = new File(mProfiler.profileFile);
                file.getParentFile().mkdirs();
                Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
            }
        } else {
        mInstrumentation = new Instrumentation();
    }
	.....
	
	// Do this after providers, since instrumentation tests generally start their
    // test thread at this point, and we don't want that racing.
        try {
            mInstrumentation.onCreate(data.instrumentationArgs);
        }
        catch (Exception e) {
            throw new RuntimeException("Exception thrown in onCreate() of "
                + data.instrumentationName + ": " + e.toString(), e);
        }
	.....
}

继续往下面执行也即是执行到didSomething |= mServices.attachApplicationLocked(app, processName);
boolean attachApplicationLocked(ProcessRecord proc, String processName)
        throws RemoteException {
	....
	for (int i=0; i<mPendingServices.size(); i++) {
        sr = mPendingServices.get(i);
    if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid || !processName.equals(sr.processName))) {
        continue;
    }
	realStartServiceLocked(sr, proc, sr.createdFromFg);
	....
}

回忆一下在上面的，以新进程的pid值作为key值保存了一个ProcessRecord在mPidsSelfLocked列表中，这里先把它取出来，存放在本地变量app中，并且将app.processName保存在本地变量processName中。
在成员变量mPendingServices中，保存了一个ServiceRecord，这里通过进程uid和进程名称将它找出来，然后通过realStartServiceLocked函数来进一步处理。
ActivityManagerService.realStartServiceLocked
这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerService.java文件中：
class ActivityManagerProxy implements IActivityManager  
{  
    ......  
  
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
          
        ......  
  
        r.app = app;  
          
        ......  
  
        try {  
  
            ......  
          
            app.thread.scheduleCreateService(r, r.serviceInfo);  
              
            ......  
  
        } finally {  
  
            ......  
  
        }  
  
        ......  
  
    }  
  
    ......  
  
}

这里的app.thread是一个ApplicationThread对象的远程接口，它是在上面创建ActivityThread对象时作为ActivityThread对象的成员变量同时创建的，也即是
final ApplicationThread mAppThread = new ApplicationThread();然后传过来的。然后调用这个远程接口的scheduleCreateService函数回到原来的ActivityThread对象中执行启动服务的操作。        
ApplicationThreadProxy.scheduleCreateService        
这个函数定义在frameworks/base/core/java/android/app/ApplicationThreadNative.java文件中  
class ApplicationThreadProxy implements IApplicationThread {  
      
    ......  
  
    public final void scheduleCreateService(IBinder token, ServiceInfo info)  throws RemoteException {  
        Parcel data = Parcel.obtain();  
        data.writeInterfaceToken(IApplicationThread.descriptor);  
        data.writeStrongBinder(token);  
        info.writeToParcel(data, 0);  
        mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null,  
            IBinder.FLAG_ONEWAY);  
        data.recycle();  
    }  
  
    ......  
  
}  

这里通过Binder驱动程序回到新进程的ApplicationThread对象中去执行scheduleCreateService函数。
ApplicationThread.scheduleCreateService
这个函数定义在frameworks/base/core/java/android/app/ActivityThread.java文件中：
public final class ActivityThread {  
      
    ......  
  
    private final class ApplicationThread extends ApplicationThreadNative {  
  
        ......  
  
        public final void scheduleCreateService(IBinder token,  
        ServiceInfo info) {  
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }  
  
        ......  
  
    }  
  
    ......  
  
}  

这里通过发送一个消息 sendMessage(H.CREATE_SERVICE, s);，这里看下消息的响应
case CREATE_SERVICE:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
    handleCreateService((CreateServiceData)msg.obj);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;

handleCreateService((CreateServiceData)msg.obj); 就会执行到
private void handleCreateService(CreateServiceData data) {
	....
	Service service = null;
        try {
			//使用类加载器加载 指定的server类
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
			
			//构建对象，然后强制转换成server对象，所以server一定要继承于server
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

		
        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
					
			//回调执行server的 onCreate方法，也就会回调到我们的server中的方法
            service.onCreate();
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
}
```