---
layout: pager
title: Android系统启动流程(五)，SystemService理解
date: 2018-01-09 21:25:59
tags: [Android,SystemService]
description:  Android系统启动流程,SystemService理解
---

Android系统启动流程,SystemService理解
<!--more-->

```java
继上一篇文章介绍 在frameworks/base/core/java/com/android/internal/os/ZygoteInit.java 的main 函数中 执行了 caller.run()，也即是执行了
mMethod.invoke(null, new Object[] { mArgs });    //执行SystemServer的main函数, 从而进入到SystemServer中  
源码的路径为 frameworks\base\services\java\com\android\server\SystemServer.java

/**
 * The main entry point from zygote.
 */
public static void main(String[] args) {
    new SystemServer().run();
}
new了一个SystemServer对象，调用自己的run方法
public SystemServer() {
        // Check for factory test mode.
        mFactoryTestMode = FactoryTest.getMode();
}

private void run() {
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

Start services. 开启服务
try {
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
} catch (Throwable ex) {
    Slog.e("System", "******************************************");
    Slog.e("System", "************ Failure starting system services", ex);
	throw ex;
} finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
    // Loop forever.
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}

createSystemContext();函数的实现为
private void createSystemContext() {
    ActivityThread activityThread = ActivityThread.systemMain();
	得到系统上下文对象
    mSystemContext = activityThread.getSystemContext();
    mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
}
ActivityThread.systemMain();函数原型
public static ActivityThread systemMain() {
    // The system process on low-memory devices do not get to use hardware
    // accelerated drawing, since this can add too much overhead to the
    // process.
    if (!ActivityManager.isHighEndGfx()) {
        ThreadedRenderer.disable(true);
    } else {
        ThreadedRenderer.enableForegroundTrimming();
    }
    自己new出对象
    ActivityThread thread = new ActivityThread();
    thread.attach(true);
    return thread;
}

startBootstrapServices();函数的原型   启动各种服务
private void startBootstrapServices() {
   
    Installer installer = mSystemServiceManager.startService(Installer.class);
		
    mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);

    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class)；
        
    mActivityManagerService.initPowerManagement();
     
    mSystemServiceManager.startService(LightsService.class);

    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

    String cryptState = SystemProperties.get("vold.decrypt");
    if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
    }

    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
				
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();

    mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
   
    AttributeCache.init(mSystemContext);
		
    mActivityManagerService.setSystemProcess();
		
        startSensorService();
}

先分析下mSystemServiceManager.startService的过程
public SystemService startService(String className) {
    final Class<SystemService> serviceClass;
    try {
		利用Class.forName将对应的类加载进来
        serviceClass = (Class<SystemService>)Class.forName(className);
    } catch (ClassNotFoundException ex) {
        Slog.i(TAG, "Starting " + className);
        throw new RuntimeException("Failed to create service " + className
                    + ": service class not found, usually indicates that the caller should "
                    + "have called PackageManager.hasSystemFeature() to check whether the "
                    + "feature is available on this device before trying to start the "
                    + "services that implement it", ex);
    }
    return startService(serviceClass);
}
 @SuppressWarnings("unchecked")
public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();
       
        final T service;
        try {
		     发射得到构造函数
             Constructor<T> constructor = serviceClass.getConstructor(Context.class);
             service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service constructor threw an exception", ex);
        }

        Register it. 将当前的服务，添加到ServiceManger中的成员变量mServices中 他的定义为 private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();
        mServices.add(service);
        // Start it.
        try {
		    启动
            service.onStart();
        } catch (RuntimeException ex) {
                throw new RuntimeException("Failed to start service " + name
                        + ": onStart threw an exception", ex);
       }
        return service;
    } finally {
       Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
   }
}

上面启动了很多系统的服务，这里分析下ActivityServiceManager启动
mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);

ActivityManagerService.Lifecycle.class 类的实现为
public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

		根据ServiceManger.startService 可知，会利用发射调用构造函数，所以这边会执行
        public Lifecycle(Context context) {
            super(context);
			构造ActivitymangerService对象
            mService = new ActivityManagerService(context);
        }

		根据ServiceManger.startService 可知  他会调用 service.onStart(); 所以这边会执行
        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
}
mService.start();也即是执行了ActivityManagerService中的start方法
private void start() {
        Process.removeAllProcessGroups();
		//cpu处理线程执行
        mProcessCpuThread.start();
        mBatteryStatsService.publish(mContext);
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
}

再来分析PackageManagerService创建过程
 mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);

public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

		//构造PackageManagerService对象
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);
        return m;
}
PackageManagerService会在构造函数的时候，就去扫描data/data/app 目录下面安装的apk的信息，然后存储起来
public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
	...
	File dataDir = Environment.getDataDirectory();
    mAppInstallDir = new File(dataDir, "app");
    mAppLib32InstallDir = new File(dataDir, "app-lib");
    mEphemeralInstallDir = new File(dataDir, "app-ephemeral");
    mAsecInternalPath = new File(dataDir, "app-asec").getPath();
    mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
	...
	扫描的方法
	scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);

    scanDirTracedLI(mDrmAppPrivateInstallDir, mDefParseFlags
                        | PackageParser.PARSE_FORWARD_LOCK,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);

    scanDirLI(mEphemeralInstallDir, mDefParseFlags
                        | PackageParser.PARSE_IS_EPHEMERAL,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);
}
关于PackageManagerService是怎么样扫描的，之后会介绍到....这里就跳过,知道是在这里启动的就好
 private void scanDirTracedLI(File dir, final int parseFlags, int scanFlags, long currentTime) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir");
        try {
            scanDirLI(dir, parseFlags, scanFlags, currentTime);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
 }


构造完之后执行  ServiceManager.addService("package", m);
ServiceManager.addService（）函数实现,可以看出这里的PackageManagerService是一个IBinder对象,查看函数的实现为 public class PackageManagerService extends IPackageManager.Stub 
可以看出PackagemanagerService是主动的将自己的IBinder对象交给了ServiceManager管理,关于IPC的，之前有分析过
public static void addService(String name, IBinder service) {
        try {
            getIServiceManager().addService(name, service, false);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }

启动核心的服务
startCoreServices();

启动其他的服务
startOtherServices();

最后调用了    Loop forever.，那此时就一直处于循环中,关于Looper的后面会介绍,这里就理解为在这边一直循环的处理处理消息
Looper.loop();
```