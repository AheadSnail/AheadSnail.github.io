---
layout: pager
title: 'Android系统启动流程(三),进入Zygote进程'
date: 2018-01-08 20:14:39
tags: [Android,Zygote]
description:  Android进入Zygote进程
---

Android 进入Zygote过程
<!--more-->

```java
继上一篇文章介绍 启动zygote就会进入framework/base/cmds/app_process/app_main.cpp的main函数在init.rc中zygote有四个参数：
-Xzygote /system/bin --zygote --start-system-server，各自的意义如下：
-Xzygote：该参数将作为虚拟机启动时所需的参数；
-/system/bin：代表虚拟机程序所在目录；
-zygote：指明以ZygoteInit.java类中的main函数作为虚拟机执行入口；
--start-system-server：告诉Zygote进程启动SystemServer进程；
在app_main.cpp的main函数中会使用这些参数。前两个参数主要对虚拟机的设置，后面的参数主要处理systemServer

zygote.rc文件的内容为
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks

在app_main.cpp的main函数中会使用这些参数。前两个参数主要对虚拟机的设置，后面的参数主要处理systemServer
#if defined(__LP64__)           //判断系统为64为还是32为, 进行赋予不同的进程名  
static const char ABI_LIST_PROPERTY[] = "ro.product.cpu.abilist64";  
static const char ZYGOTE_NICE_NAME[] = "zygote64";  
#else  
static const char ABI_LIST_PROPERTY[] = "ro.product.cpu.abilist32";  
static const char ZYGOTE_NICE_NAME[] = "zygote";  
#endif  

main函数的主要代码为
int main(int argc, char* const argv[])
{
	......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}

一上来就构建AppRuntime对象，就会执行对应的构造函数
AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
可以发现AppRuntime是继承于AndroidRuntime的
class AppRuntime : public AndroidRuntime
{
public:
    AppRuntime(char* argBlockStart, const size_t argBlockLength)
        : AndroidRuntime(argBlockStart, argBlockLength)//执行父类的构造函数，也就是AndroidRuntime.cpp的构造函数
        , mClass(NULL)
    {
    }
}


AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),//用成员变量接收过来
        mArgBlockLength(argBlockLength)//用成员变量接收过来
{
// Pre-allocate enough space to hold a fair number of options.  
    mOptions.setCapacity(20);//mOptions 的函数原型为     Vector<JavaVMOption> mOptions; 为一个向量集合,这里设置这个集合的容量
}

构造完对象之后，mian函数继续往下面执行，将传递的参数，添加到runtime对象中
 int i;
 for (i = 0; i < argc; i++) {
     if (argv[i][0] != '-') {
        break;
    }
    if (argv[i][1] == '-' && argv[i][2] == 0) {
        ++i; // Skip --.
        break;
    }
    runtime.addOption(strdup(argv[i]));
}

void AndroidRuntime::addOption(const char* optionString, void* extraInfo)
{
	//构建Option对象
    JavaVMOption opt;
    opt.optionString = optionString;
    opt.extraInfo = extraInfo;
	//将option添加到向量结合mOptions中保存起来
    mOptions.add(opt);
}


main函数继续执行
// Parse runtime arguments.  Stop at first unrecognized option. //解析运行时的参数,从解析init.rc中获得  
bool zygote = false;  
bool startSystemServer = false;  
bool application = false;  
String8 niceName;  
String8 className;  
  
++i;  // Skip unused "parent dir" argument.  
while (i < argc) {  
    const char* arg = argv[i++];  
    if (strcmp(arg, "--zygote") == 0) {  
        zygote = true;    //在zygote模式中启动  
        niceName = ZYGOTE_NICE_NAME; //设置进程的名字  
    } else if (strcmp(arg, "--start-system-server") == 0) {  
        startSystemServer = true;   //启动System Server  
    } else if (strcmp(arg, "--application") == 0) {  
        application = true;          //在application模式启动  
    } else if (strncmp(arg, "--nice-name=", 12) == 0) {  
        niceName.setTo(arg + 12);     //参数中有进程名,就设置  
    } else if (strncmp(arg, "--", 2) != 0) {  
        className.setTo(arg);   //设置className  
        break;  
    } else {  
        --i;  
        break;  
    }  
}  

mian函数继续往下面执行
Vector<String8> args;//定义一个向量的集合用来传递参数
if (!className.isEmpty()) {
    // We're not in zygote mode, the only argument we need to pass
    // to RuntimeInit is the application argument.
    //
    // The Remainder of args get passed to startup class main(). Make
    // copies of them before we overwrite them with the process name.
    args.add(application ? String8("application") : String8("tool"));
    runtime.setClassNameAndArgs(className, argc - i, argv + i);
} else {
	根据我们传递的参数会进入到这里
    // We're in zygote mode.
    maybeCreateDalvikCache();  //创建data/dalvik-cache,设置环境 

    if (startSystemServer) {  //startSystemServer为true  
        args.add(String8("start-system-server"));//会添加这个参数
    }

    char prop[PROP_VALUE_MAX];
    if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
        LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
            ABI_LIST_PROPERTY);
        return 11;
    }

    String8 abiFlag("--abi-list=");
    abiFlag.append(prop);
    args.add(abiFlag);//再添加这个参数

    // In zygote mode, pass all remaining arguments to the zygote
    // main() method.
	//解析所有传递的参数，添加到args中
    for (; i < argc; ++i) {
        args.add(String8(argv[i]));
    }
}

main函数继续执行 
if (!niceName.isEmpty()) {//通过上面参数的解析，知道niceName 是不为空的,所以会执行下面的代码   niceName = ZYGOTE_NICE_NAME; static const char ZYGOTE_NICE_NAME[] = "zygote";
    runtime.setArgv0(niceName.string());
    set_process_name(niceName.string());
}

void AndroidRuntime::setArgv0(const char* argv0) {
    memset(mArgBlockStart, 0, mArgBlockLength); //mArgBlockStart 函数原型为  char* const mArgBlockStart;这个成员变量在上面构造函数中已经赋值了，所以这里不会为野指针,重置这块内存空间
    strlcpy(mArgBlockStart, argv0, mArgBlockLength);//将zygote 赋值给mArgBlockStart 这块内存空间
}

set_process_name(niceName.string());为设置当前进程的名字，也即为zygote进程,系统的函数
 
main函数继续执行
if (zygote) {//zygote在上面的解析参数中设置了 zygote = true;   标识在zygote模式中启动  所以会执行下面的代码
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
} else if (className) {
    runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
} else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    return 10;
}

执行下面的代码,就会执行到AndroidRuntime.cpp中对应的start函数
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)// runtime.start("com.android.internal.os.ZygoteInit", args, true);
{
	........
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {//启动虚拟机  ,并构建mJavaVM JNIEnv对象
        return;
    }
	
    onVmCreated(env);//对虚拟机进行初始化  

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

	//清理mJavaVM对象
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}

startVm(&mJavaVM, &env, zygote)启动虚拟机的函数所在的位置就在AndroidRuntime.cpp中的startVm中，因为比较复杂，这里就不看了
/*
 * Start the Dalvik Virtual Machine.
 *
 * Various arguments, most determined by system properties, are passed in.
 * The "mOptions" vector is updated.
 *
 * CAUTION: when adding options in here, be careful not to put the
 * char buffer inside a nested scope.  Adding the buffer to the
 * options using mOptions.add() does not copy the buffer, so if the
 * buffer goes out of scope the option may be overwritten.  It's best
 * to put the buffer at the top of the function so that it is more
 * unlikely that someone will surround it in a scope at a later time
 * and thus introduce a bug.
 *
 * Returns 0 on success.
 */
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
	....
	initArgs.version = JNI_VERSION_1_4;
    initArgs.options = mOptions.editArray();
    initArgs.nOptions = mOptions.size();
    initArgs.ignoreUnrecognized = JNI_FALSE;

    /*
     * Initialize the VM.
     *
     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
     * If this call succeeds, the VM is ready, and we can start issuing
     * JNI calls.
     */
	 //调用JNI_CreateJavaVM函数，完成pJavaVM，pEnv的赋值,此函数为jni.h的提供的函数
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
}

start函数继续执行

jclass stringClass;
jobjectArray strArray;
jstring classNameStr;

stringClass = env->FindClass("java/lang/String");//找到string类
assert(stringClass != NULL);
strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);//构建一个java对应的字符串数组
assert(strArray != NULL);
classNameStr = env->NewStringUTF(className);//className 为 com.android.internal.os.ZygoteInit，也即是将c中的字符串变为java的字符串
assert(classNameStr != NULL);
env->SetObjectArrayElement(strArray, 0, classNameStr);//将转换之后的classNameStr字符串填充到java 字符数组的第一个元素，字符的内容为com.android.internal.os.ZygoteInit

//之后将options中的参数，全部由c中的字符串变成java中的字符串，然后填充到java数组中
for (size_t i = 0; i < options.size(); ++i) {
    jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
    assert(optionsStr != NULL);
    env->SetObjectArrayElement(strArray, i + 1, optionsStr);
}

/*
* Start VM.  This thread becomes the main thread of the VM, and will
* not return until the VM exits.
*/
char* slashClassName = toSlashClassName(className);
jclass startClass = env->FindClass(slashClassName);//调用jni的方法，得到com.android.internal.os.ZygoteInit对应的class对象
if (startClass == NULL) {//如果为空，就代表找不到，这个包名对应的类
     ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
    /* keep going */
} else {
	//找到了com.android.internal.os.ZygoteInit 类
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main","([Ljava/lang/String;)V");//找到对应类中的main函数
    if (startMeth == NULL) {//找不到main函数，就会返回空
        ALOGE("JavaVM unable to find main() in '%s'\n", className);
         /* keep going */
    } else {
		//找到了对应的main函数,最终利用jni的函数CallStaticVoidMethod，回调执行到java层的main函数
		//ZygoteInit代码位置frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
        env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
        if (env->ExceptionCheck())
           threadExitUncaughtException(env);
#endif
        }
    }
free(slashClassName);

```

在调用完ZygoteInit的main函数后，Zygote就进入了Java世界。
![结果显示](/uploads/Android启动流程三/zygoteInit路径.png)

```java
ZygoteInit代码位置frameworks/base/core/java/com/android/internal/os/ZygoteInit.java , 
在ZygoteInit的main函数中首先将从底层中传过来的参数获取出来.

  mian函数的代码为
  public static void main(String argv[]) {
		......
        try {
            ......
            boolean startSystemServer = false;
            String socketName = "zygote"; //socketName为zygote  
            String abiList = null;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) { //start-system-server在参数列表中,故为true   在C层中 args.add(String8("start-system-server"));有添加这个参数
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {//设置abiList  
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {//如果参数列表有socketName,就重新设置socketName  
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            registerZygoteSocket(socketName);//在ZygoteInit中通过RegisterZygoteSocket()进行注册Socket，用于进程间通信。
				
            preload();//Zygote 执行预加载
        
            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PostZygoteInitGC");
            gcAndFinalize();
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);

            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            // Zygote process unmounts root storage spaces.
            Zygote.nativeUnmountStorageOnInit();

            ZygoteHooks.stopZygoteNoThreadCreation();

            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }

            Log.i(TAG, "Accepting command socket connections");
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (Throwable ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
	
registerZygoteSocket(socketName);函数的实现为
/**
 * Registers a server socket for zygote command connections
*
* @throws RuntimeException when open fails
*/
private static void registerZygoteSocket(String socketName) {
if (sServerSocket == null) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;//设置完整的socketName  
        try {
            String env = System.getenv(fullSocketName);//获取环境变量  
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(fullSocketName + " unset or invalid", ex);
        }

        try {
            FileDescriptor fd = new FileDescriptor();
            fd.setInt$(fileDesc);//设置文件描述符  
            sServerSocket = new LocalServerSocket(fd); //根据文件描述符创建Server Socket  
        } catch (IOException ex) {
            throw new RuntimeException("Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}

new LocalServerSocket(fd)函数的原型为
public LocalServerSocket(FileDescriptor fd) throws IOException
{
    impl = new LocalSocketImpl(fd);
    impl.listen(LISTEN_BACKLOG);//采用Linux系统的中的select函数监听Socket文件描述符
	localAddress = impl.getSockAddress();
}

其中Socket的监听方式为使用Linux系统调用select()函数监听Socket文件描述符，当该文件描述符上有数据时，自动触发中断，在中断处理函数中去读取文件描述符中的数据。

preload();//Zygote 执行预加载，至于为什么要执行预加载，下面为解释
在Android系统中有很多的公共资源，所有的程序都会需要。而Zygote创建应用程序进程过程其实就是复制自身进程地址空间作为应用程序进程的地址空间，
因此在Zygote进程中加载的类和资源都可以共享给所有由Zygote进程孵化的应用程序。所以就可以在Zygote中对公共类与资源进行加载，当应用程序启动时只需要加载自身特有的类与资源就行了，
提高了应用软件的启动速度，这也看出面向对象编程中继承关系的好处。所以

预加载的函数实现为
static void preload() {
        Log.d(TAG, "begin preload");
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "BeginIcuCachePinning");
        beginIcuCachePinning();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadClasses");
        preloadClasses();//预加载Classes  
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadResources");
        preloadResources();//预加载resources  
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadOpenGL");
        preloadOpenGL();//预加载openGL  
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        preloadSharedLibraries();//预加载共享类库
        preloadTextResources(); //预加载文本资源 
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        endIcuCachePinning();
        warmUpJcaProviders();
        Log.d(TAG, "end preload");
}

```

需要预加载的classes在手机中system/etc/preloaded-classes里面可以看到
![结果显示](/uploads/Android启动流程三/zygote预加载类.png)


```java
 /** 
 * The path of a file that contains classes to preload. 
 */  
private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes";  
  
//预加载classes
private static void preloadClasses() {  
    final VMRuntime runtime = VMRuntime.getRuntime();  
    InputStream is;  
    try {  
            is = new FileInputStream(PRELOADED_CLASSES);   //将classes文件转变为文件输入流  
        } catch (FileNotFoundException e) {  
            Log.e(TAG, "Couldn't find " + PRELOADED_CLASSES + ".");  
            return;  
        }  
  
        Log.i(TAG, "Preloading classes...");  
        long startTime = SystemClock.uptimeMillis();   //开始预加载时间  
        //.........  
        try {  
            BufferedReader br  
                = new BufferedReader(new InputStreamReader(is), 256);   //封装成BufferReader  
  
            int count = 0;  
            String line;  
            while ((line = br.readLine()) != null) {   
                // Skip comments and blank lines.  
                line = line.trim();  
                if (line.startsWith("#") || line.equals("")) {  //将没有用的行过滤掉   也就是前面带有
                    continue;  
                }  
  
                Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadClass " + line);  
                try {  
                    if (false) {  
                        Log.v(TAG, "Preloading " + line + "...");  
                    }  
                    // Load and explicitly initialize the given class. Use  
                    // Class.forName(String, boolean, ClassLoader) to avoid repeated stack lookups  
                    // (to derive the caller's class-loader). Use true to force initialization, and  
                    // null for the boot classpath class-loader (could as well cache the  
                    // class-loader of this class in a variable).  
                    Class.forName(line, true, null);   //要求JVM查找并加载指定的类
                    count++;  
                } catch (ClassNotFoundException e) {  
                    Log.w(TAG, "Class not found for preloading: " + line);  
                } catch (UnsatisfiedLinkError e) {  
                    Log.w(TAG, "Problem preloading " + line + ": " + e);  
                } catch (Throwable t) {  
                    Log.e(TAG, "Error preloading " + line + ".", t);  
                    if (t instanceof Error) {  
                        throw (Error) t;  
                    }  
                    if (t instanceof RuntimeException) {  
                        throw (RuntimeException) t;  
                    }  
                    throw new RuntimeException(t);  
                }  
                Trace.traceEnd(Trace.TRACE_TAG_DALVIK);  
            }  
  
            Log.i(TAG, "...preloaded " + count + " classes in "  
                    + (SystemClock.uptimeMillis()-startTime) + "ms.");  //计算预加载classes一共花了多长时间  
  //......  
}  

//预加载资源文件
private static void preloadResources() {  
    final VMRuntime runtime = VMRuntime.getRuntime();  
  
    try {  
        mResources = Resources.getSystem();  
        mResources.startPreloading();  
        if (PRELOAD_RESOURCES) {  
			Log.i(TAG, "Preloading resources...");  
  
            long startTime = SystemClock.uptimeMillis();  //记录开始预加载resources时间点  
            TypedArray ar = mResources.obtainTypedArray(   //所需要加载的资源，其实就是在framework/base/core/res/res/values/arrays.xml中进行定义的  
                      com.android.internal.R.array.preloaded_drawables);  //获取需要预加载的drawables  
            int N = preloadDrawables(ar);      //预加载drawables  
            ar.recycle();  
            Log.i(TAG, "...preloaded " + N + " resources in "  
                      + (SystemClock.uptimeMillis()-startTime) + "ms.");   //预加载drawables耗时情况  
  
            startTime = SystemClock.uptimeMillis();     //开始预加载colorState时间点  
            ar = mResources.obtainTypedArray(  
                      com.android.internal.R.array.preloaded_color_state_lists);   //获取预加载的color state lists  
            N = preloadColorStateLists(ar);    //预加载color state  
            ar.recycle();  
            Log.i(TAG, "...preloaded " + N + " resources in "  
                      + (SystemClock.uptimeMillis()-startTime) + "ms.");  //预加载resoures耗时  
  
            if (mResources.getBoolean(  
                      com.android.internal.R.bool.config_freeformWindowManagement)) {  
                  startTime = SystemClock.uptimeMillis();  
                  ar = mResources.obtainTypedArray(  
                          com.android.internal.R.array.preloaded_freeform_multi_window_drawables);  
                  N = preloadDrawables(ar);  
                  ar.recycle();  
                  Log.i(TAG, "...preloaded " + N + " resource in "  
                          + (SystemClock.uptimeMillis() - startTime) + "ms.");  
             }  
        }  
        mResources.finishPreloading();  
    } catch (RuntimeException e) {  
          Log.w(TAG, "Failure preloading resources", e);  
    }  
}  
```

com.android.internal.R.array.preloaded_drawables的资源文件的内容
![结果显示](/uploads/Android启动流程三/zygote预加载资源的内容.png)

com.android.internal.R.array.preloaded_drawables 预加载资源文件的路径
![结果显示](/uploads/Android启动流程三/预加载资源文件的路径.png)

com.android.internal.R.array.preloaded_color_state_lists 的资源文件的内容
![结果显示](/uploads/Android启动流程三/zygote预加载color内容.png)

com.android.internal.R.array.preloaded_color_state_lists的资源路径跟上面的com.android.internal.R.array.preloaded_drawables是同一个文件

```java
//预加载OpgL
private static void preloadOpenGL() {
        if (!SystemProperties.getBoolean(PROPERTY_DISABLE_OPENGL_PRELOADING, false)) {
            EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
        }
}

//预加载共享库
private static void preloadSharedLibraries() {
        Log.i(TAG, "Preloading shared libraries...");
        System.loadLibrary("android");
        System.loadLibrary("compiler_rt");
        System.loadLibrary("jnigraphics");
}
	
//预加载资源文本
 private static void preloadTextResources() {
        Hyphenator.init();
        TextView.preloadFontCache();
}

继续往下执行,就该创建SystemServer, 在后文中将讲解.
并且循环进入监听模式, 监听创建进程的请求
if (startSystemServer) {   //startSysteServer为true  
    startSystemServer(abiList, socketName);     
}  
  
Log.i(TAG, "Accepting command socket connections");  
runSelectLoop(abiList);     //接受应用的socket链接请求  
closeServerSocket();   //当退出循环时,系统出现问题, 所以也要将server socket关闭.  


之后Zygote等待客户端的创建进程请求.
/** 
 * Runs the zygote process's select loop. Accepts new connections as 
 * they happen, and reads commands from connections one spawn-request's 
 * worth at a time. 
 * 
 * @throws MethodAndArgsCaller in a child process when a main() should 
 * be executed. 
 */  
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {  
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();  
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();  
  
    fds.add(sServerSocket.getFileDescriptor());   
    peers.add(null);  
  
    while (true) {  
        StructPollfd[] pollFds = new StructPollfd[fds.size()];  
        for (int i = 0; i < pollFds.length; ++i) {  
            pollFds[i] = new StructPollfd();  
            pollFds[i].fd = fds.get(i);  
            pollFds[i].events = (short) POLLIN;  
        }  
        try {  
            Os.poll(pollFds, -1);  
        } catch (ErrnoException ex) {  
            throw new RuntimeException("poll failed", ex);  
        }  
        for (int i = pollFds.length - 1; i >= 0; --i) {  
            if ((pollFds[i].revents & POLLIN) == 0) {  
                continue;  
            }  
            if (i == 0) {  
                ZygoteConnection newPeer = acceptCommandPeer(abiList);  
                peers.add(newPeer);  
                fds.add(newPeer.getFileDesciptor());  
            } else {  
                boolean done = peers.get(i).runOnce();  
                if (done) {  
                    peers.remove(i);  
                    fds.remove(i);  
                }  
            }  
        }  
    }  
}  
```
流程总结
![结果显示](/uploads/Android启动流程三/zygote启动进入到zygoteinit流程图.jpg)